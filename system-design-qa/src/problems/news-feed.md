# Q: Design Twitter / a news feed.

**Answer:**

Canonical "high read, fan-out" problem. The interesting tension is **fan-out on write** (push to followers' feeds) vs **fan-out on read** (pull from followees' tweets). A real system uses both selectively.

### Requirements

**Functional**:
- Post tweets (text, ~280 chars).
- Follow / unfollow users.
- Home timeline: chronological (or ranked) feed of followed users' tweets.
- User timeline: tweets by one user.

**Non-functional**:
- 200M DAU.
- Avg 1 post / user / day, 100 reads / user / day → 230k QPS reads, 2.3k QPS writes.
- p99 timeline load < 200 ms.
- Some users have 100M followers (celebrities).

### Capacity Estimation

**Tweets per day**: 200M.
**Storage per tweet**: ~300 B (text + metadata).
**Daily growth**: 60 GB; 22 TB/year text only.

**Home timeline reads**: 230k QPS. Each loads ~50 tweets. → ~12M tweet-reads/sec.

### Fan-Out Strategies

**Push (fan-out on write)**

```
On tweet:
  for each follower:
    insert tweet_id into follower's timeline

On read: just SELECT * FROM timelines WHERE user_id = X ORDER BY ts DESC LIMIT 50
```

Read is trivial. Write multiplies by followers.

**Pull (fan-out on read)**

```
On tweet: insert into tweets table only.

On read:
  for each user I follow:
    fetch their latest tweets
  merge, sort, paginate
```

Write is cheap. Read fetches from many sources.

**The hybrid (Twitter actual)**

- **Push** for typical users (followers in thousands).
- **Pull** for celebrities (followers in millions/100M+).
- On timeline read: merge pre-built timeline + last-tweets from celebrities the user follows.

Why: pushing one Justin Bieber tweet to 100M followers is a 100M-row insert. Untenable.

### Data Model

**Tweets** (write-once, key by tweet_id):

```
tweets {
    id            BIGINT (snowflake)
    user_id       BIGINT
    body          TEXT
    created_at    TIMESTAMPTZ
    parent_id     BIGINT NULL  (for replies)
}
```

Store in Cassandra/DynamoDB partitioned by `user_id` + `created_at` for user-timeline reads.

**Follows**:

```
followers {
    user_id        BIGINT  (who is being followed)
    follower_id    BIGINT
    created_at     TIMESTAMPTZ
}
following {
    user_id        BIGINT  (who is doing the following)
    followed_id    BIGINT
}
```

Two tables for O(1) lookup in both directions.

**Timeline cache** (Redis sorted set per user):

```
timeline:user_42 = ZSET of (tweet_id, score=timestamp)
```

Cache last ~800 entries (covering ~weeks for typical users).

### Architecture

```
                  Client (mobile, web)
                          │
                          ▼
                       LB / CDN
                          │
            ┌─────────────┼──────────────┐
            ▼             ▼              ▼
        Write API    Read API       User API
            │             │              │
            ▼             ▼              ▼
       Tweet Service  Timeline       User / Follow
            │         Service         Service
            │             │              │
            ▼             ▼              ▼
       Cassandra      Redis           Postgres
                       (timelines)
                          ▲
                          │
            Fan-out worker (Kafka consumer)
                          ▲
                          │
       Kafka topic "tweets-posted"
                          ▲
                          │
                     Tweet Service
```

### Write Path

```
POST /tweet
  1. INSERT INTO tweets (id, user_id, body, created_at)
  2. PUBLISH to Kafka topic "tweets-posted"
  3. Return 201 to user

Async fan-out worker:
  consumer reads "tweets-posted"
  for each tweet:
    is the author a celebrity?  (> 1M followers)
       yes: skip push (pull path will fetch)
       no:  for each follower:
              ZADD timeline:user_<follower> <ts> <tweet_id>
              ZREMRANGEBYRANK timeline:user_<follower> 0 -801   (cap at 800)
```

### Read Path

```
GET /home
  1. tl = ZREVRANGE timeline:user_<me> 0 49
  2. for each celebrity I follow:
        latest = fetch their tweets > my last_seen
        merge into tl
  3. fetch tweet bodies for each tweet_id (multi-get from cache or Cassandra)
  4. return sorted by timestamp
```

Typical user follows 0–10 celebrities; the merge is fast.

### Hot Spots

**Celebrity tweet**: 100M followers. Fan-out push would crush the system. Hybrid pull path handles it.

**Celebrity timeline read** (visiting Elon's profile): high QPS on one partition. Cache aggressively at CDN/edge.

**Viral tweet**: one tweet getting millions of reads. Standard hot-key problem; cache + CDN.

### Pagination / Cursors

Don't use offset-pagination at scale. Use cursor:

```
GET /home?cursor=<last_tweet_id>
```

Cursor encodes `(timestamp, tweet_id)`. ZRANGEBYSCORE in Redis.

### Storage Decisions

| Data | Store | Why |
|------|-------|-----|
| Tweets | Cassandra | Massive write throughput, partition by user_id |
| Timelines | Redis (ZSET) | Sorted, fast, capped size |
| Follows | DynamoDB / Postgres | Frequent both-direction lookup |
| User profile | Postgres | ACID, identity |
| Search | Elasticsearch | Text search, hashtag |
| Trends, counts | ClickHouse / Druid | Aggregations, time windows |
| Media | S3 | Photos, video |

### Search / Hashtag

Indexed separately in Elasticsearch (or a tweet ingestion pipeline → search index).

```
on tweet posted:
  → publish to Kafka
  → indexer consumer:
       parse hashtags, mentions
       index into Elasticsearch (text, tags, user, ts)
```

Search QPS handled by ES cluster, independent of timeline path.

### Ranking (Optional)

Pure chronological is the baseline. Ranked feed:

- Score each candidate tweet (recency × author affinity × engagement).
- ML model serves scores; ranker picks top N.
- Pre-rank candidates offline; rerank online with light model.

### Failure Modes

| Failure | Handling |
|---------|---------|
| Fan-out worker behind | Lag increases; user sees stale timeline. Add workers. |
| Redis timeline cache cold | Reconstruct from Cassandra on demand; cache warm. |
| One Cassandra node down | Replication factor 3, quorum reads. |
| Celebrity user posts during outage | Fan-out delayed; pull path still works. |
| Tweet duplicated on retry | Idempotency via client-provided tweet_id. |

### Multi-Region

- **Tweets**: globally replicated (Cassandra cross-DC).
- **Timelines**: regional (rebuilt from tweet stream).
- **Follows**: globally replicated; eventual.

Trade: a follow done in EU shows up in US within seconds. Acceptable.

### Trade-offs Discussion

**Cost of push fan-out**:
- 200M users × avg ~200 followers = 40B inserts/day = 460k inserts/sec into timelines.
- Mitigated by: skipping celebrities, batching, write-back caching, capping timeline size.

**Cost of pull fan-out**:
- 230k home reads × N followees per user (avg 200) = 46M backend reads/sec.
- Untenable without aggressive caching.

The hybrid is the right answer because real social graphs are heavy-tailed: median user has few followees, celebrities have millions of followers.

### Extensions

- **Read receipts / seen indicators** (Snapchat-like): per-user-per-tweet state — denormalized, eventually consistent.
- **Threading / replies**: tweet has `parent_id`; threaded view fetches whole conversation.
- **Notifications**: separate service triggered on mention/follow/retweet. See [Notification System](./notification-system.md).
- **Trends**: HLL + Count-Min Sketch over the tweet stream for real-time top hashtags.

### Common Mistakes

| Mistake | Fix |
|---------|-----|
| Pure push or pure pull | Production is hybrid |
| Storing timelines in RDBMS | At this scale, no |
| Synchronous fan-out in the write path | Latency unacceptable; always async |
| One celebrity's followers all push together | Worker queue; deduplicate work |
| Offset pagination | Breaks past page 50 |

> [!NOTE]
> The defining insight is **graph asymmetry**. A pull-only system is wrong for the median user; a push-only system is wrong for celebrities. Pick the right path per author.

### Interview Follow-ups

- *"How do you handle a flood of tweets from one user?"* — Per-user rate limit; spam detection; tier celebrities into separate fan-out paths.
- *"How do you do real-time updates (live timeline)?"* — Long-lived connection (WebSocket / SSE); push tweet IDs as they arrive; client prepends.
- *"How would unfollow work?"* — Remove from `follows`; lazy-cleanup of stale tweets in timeline (or live filter on read using current follow set).
