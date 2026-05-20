# Q: Design a distributed counter (likes, views, ad impressions).

**Answer:**

Looks trivial; gets hard fast. The naive `UPDATE counter += 1` doesn't scale past a few thousand QPS on one row. Real systems use **sharded counters**, **batching**, or **CRDTs**.

### Requirements

**Functional**:
- Increment / decrement.
- Read current value.
- Multiple counters (per video, per ad, per user).

**Non-functional**:
- 1M increments/sec aggregate, spikes on viral content.
- Read latency < 100 ms.
- Approximate is OK for views/likes; exact for billing.

### Why It's Hard

```
UPDATE videos SET views = views + 1 WHERE id = X;
```

At high QPS this single row becomes a:
- Row-lock contention point.
- Write-throughput bottleneck.
- Replication lag amplifier.

A celebrity's video at 100k views/sec destroys a Postgres row.

### Approach 1: Sharded Counter

Split one logical counter into N physical sub-counters:

```
counter[X] is the sum of:
  counter:X:0
  counter:X:1
  ...
  counter:X:99

increment: pick a random shard, += 1
read: sum all shards
```

Write QPS / N per shard. With N=100, one shard takes 1% of writes.

Read more expensive: `SELECT sum(c) FROM shards WHERE id = X`. Cache the sum.

Used by Google App Engine, many large-scale counters.

### Approach 2: Batched Increment

Coalesce increments locally before writing:

```
in-process buffer:
  ++local_count[X]
  every 100 ms, flush:
    UPDATE counter SET v = v + local_count[X] WHERE id = X
    local_count[X] = 0
```

One write per 100 ms × #processes. Cuts DB load 100×+.

Downside: brief inconsistency (100ms lag). Acceptable for views/likes.

### Approach 3: Stream-Based (Kafka)

```
events ──► Kafka topic ──► consumer ──► +N to DB row (batched)
                            │
                            ▼
                       hot cache (Redis) for reads
```

Producer fires `+1` event; consumer aggregates by key in a tumbling window; writes increment.

Decouples write path from DB throughput. Lag observable as Kafka consumer lag.

### Approach 4: Redis with Periodic Sync

Hot writes go to Redis (`INCR`), periodically flushed to durable DB:

```
on increment:    INCR counter:X       (~100k ops/sec on Redis)
on read:         GET counter:X

every minute, flush dirty keys to Postgres
```

Redis is the source of truth for "current"; Postgres is durable backup.

Risk: Redis crash loses the unflushed deltas. Mitigate with AOF persistence or counter snapshots.

### Approach 5: CRDT G-Counter

For multi-region active-active where each region writes independently:

```
state per region: { region1: 100, region2: 50, region3: 25 }
local increment: increment own region's slot
total = sum across regions
merge two states: max per region
```

G-Counters merge mathematically — no conflict possible. Used for "likes," "views" across regions.

Trade: can't decrement (use PN-Counter — pair of G-Counters for + and −).

### Hybrid Production Pattern

Combine the above:

```
client increment ─► gateway ─► Kafka ─► aggregator window (10s) ─► Redis (hot)
                                                                      │
                                                          every 1 min flush
                                                                      ▼
                                                                  Postgres
```

- Hot reads from Redis, < 1 ms.
- Eventual durability via Postgres flush.
- Multi-region via per-region aggregator + CRDT merge if needed.

### Choosing By Use Case

| Use case | Approach |
|----------|----------|
| Per-post like count (huge fanout, approximate OK) | Sharded counter + cache |
| Page view count (huge volume, eventual) | Stream aggregator + Redis |
| Ad impression count (billing, must be exact) | Stream with exactly-once dedup + reconciliation |
| Distributed user score | CRDT G-Counter |
| Rate-limit counter (per second) | Redis INCR with TTL |

### Read Semantics

| Read type | Mechanism |
|-----------|----------|
| Approximate current | Cache (Redis); refresh from DB lazily |
| Up-to-the-millisecond exact | Read all shards + sum; expensive |
| Eventually exact | Background aggregator computes; cache result |

Decide what users actually need. "1.2K likes" doesn't need to be exact.

### Negative Increments

Decrement (unlike, retract impression):

- For sharded counter: `--` on same shard the original was on (track) or random (acceptable for approximate).
- CRDT: separate P (positive) and N (negative) counters; result = sum(P) - sum(N).

### Idempotency

Network retries → same event twice → double-counting.

- Each event has a unique ID.
- Aggregator dedups by ID before incrementing.
- DB upsert "if not seen, +=N."

For ad billing, deduplication is mandatory and audited.

### Time-Windowed Counters

"Views in the last 5 minutes":

- Sliding window over event stream (Flink, Kafka Streams).
- Or HyperLogLog if approximate unique count is enough.

### Top-K Counters

"Top 10 trending videos":

- Each video has counter.
- Top-K maintained via Count-Min Sketch or sorted set (Redis ZSET).
- Refresh window: 1 min for "trending," real-time for live counts.

### Failure Modes

| Failure | Handling |
|---------|---------|
| Redis crash before flush | Lose ~minute of increments (acceptable for likes; not for billing) |
| Aggregator backed up | Counter lag grows; alert |
| One shard hot (skew across shards) | Add more shards or rebalance |
| Spam ("like bot") | Deduplicate by user_id at producer; rate-limit |

### Common Mistakes

| Mistake | Fix |
|---------|-----|
| Direct `UPDATE` for hot counter | Throughput ceiling hit fast |
| One row per like (no aggregation) | DB hammered; aggregate in stream |
| No batching | Each event a transaction; throughput tanks |
| Cache without invalidation policy | Stale values for hours |
| Ignoring multi-region | LWW destroys counts; use CRDT or single owner |

> [!NOTE]
> Distributed counters are a perfect example of "what looks like a one-line problem requires a 100-line architecture at scale." The right answer depends on whether you need exact or approximate, real-time or eventual, single-region or global.

### Interview Follow-ups

- *"How do you keep ad impression counts exact?"* — Idempotent ingestion (event_id dedup); end-of-day reconciliation; double-entry-style audit log.
- *"What about decrement when the cache lag is unknown?"* — Track positives and negatives separately (G-Counter / PN-Counter); merge correctly.
- *"How would you show 'live view count' on a video?"* — Sliding window over Kafka events; push deltas to clients via WebSocket every few seconds.
