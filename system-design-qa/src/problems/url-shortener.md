# Q: Design a URL shortener (bit.ly).

**Answer:**

Classic interview opener. Looks trivial; the depth comes from key generation, scale, and analytics. Walk through it as a real production design.

### Requirements

**Functional**:
- Shorten a long URL → 7-character short code.
- Resolve short code → 301/302 redirect to long URL.
- Custom aliases (optional).
- Expiration (optional).
- Click analytics (optional).

**Non-functional**:
- 200M shortenings/day → ~2,300 writes/sec.
- 20B reads/day → ~230,000 reads/sec.
- Read:write ≈ 100:1.
- p99 redirect latency < 50 ms (it's in the user's browser path).
- 5+ year retention.
- High availability — broken short link = broken landing pages.

### Capacity Estimation

**Storage**:

```
Per record:
  short code (7 char)         ~7 B
  long URL (avg 200 char)   ~200 B
  metadata (created, owner, ...) ~100 B
  indexes overhead          ~150 B
  Total                    ~450 B
   per day:    200M × 450 B = 90 GB
   per year:   33 TB
   5 years:    ~165 TB
```

**Bandwidth**:

```
Redirect QPS: 230k
Response: ~500 B headers
   230k × 500 B ≈ 115 MB/s
```

Trivially served from a handful of well-cached frontends.

### Short Code Design

7 characters in `[a-zA-Z0-9]` = 62^7 ≈ 3.5 × 10¹². Plenty.

Three generation strategies:

**1. Random + collision check.**

```
loop:
  code = random(62, 7)
  if not exists(code) → atomically insert
```

Pros: stateless, parallel-friendly.
Cons: collisions grow with table size; needs unique-constraint insert.

**2. Hash of URL.**

```
code = base62(sha256(url))[:7]
```

Pros: deterministic; same URL → same code.
Cons: collisions (different URLs → same 7-char hash); same URL by two users gets the same code (sometimes desired, sometimes not).

**3. Counter + base62 encoding.**

```
id = counter.increment()       # global monotonic
code = base62(id)
```

Pros: zero collisions, ordered.
Cons: global counter = single point. Solve with:
- Distributed counter (etcd / ZooKeeper).
- Pre-allocated batches: each app instance gets ranges of 10k IDs.
- Snowflake-style 64-bit IDs split into machine + sequence.

**Production pick**: pre-allocated batches per instance + base62. Simple, scalable, predictable.

### API

```
POST /v1/shorten
  body: { url, custom_alias?, expires_at? }
  resp: { short_url: "https://sho.rt/abc1234" }

GET /v1/{code}
  resp: 301 Location: <long_url>

GET /v1/{code}/stats
  resp: { clicks, geo, browsers, ... }
```

Idempotency: `Idempotency-Key` so client retries don't create duplicate codes.

### Data Model

```sql
CREATE TABLE urls (
    code        VARCHAR(10) PRIMARY KEY,
    long_url    TEXT NOT NULL,
    user_id     BIGINT,
    created_at  TIMESTAMPTZ DEFAULT now(),
    expires_at  TIMESTAMPTZ,
    -- consider: INDEX on (user_id, created_at)
);

CREATE TABLE clicks (
    code        VARCHAR(10),
    clicked_at  TIMESTAMPTZ,
    ip          INET,
    user_agent  TEXT,
    referer     TEXT
);
```

For 5-year × 230k QPS reads, Postgres alone can't serve from disk. We need caching.

### High-Level Architecture

```
                                  ┌──── CDN (cache 30s)
Browser ───────────────────────►  │
                                  ▼
                              LB (ALB / Cloudflare)
                                  │
                                  ▼
                            Stateless API
                              /      \
                             ▼        ▼
                       Redis cache   Counter service
                            │ miss
                            ▼
                     Primary DB (Postgres)
                            │ async
                            ▼
                         Kafka ── click events ──► Analytics (ClickHouse)
                            │
                            ▼
                       S3 cold archive
```

### Read Path

Most reads are repeat redirects to the same popular URLs (Pareto):

1. CDN cache for popular codes (TTL ~30s).
2. Redis: `GET code` → return long URL.
3. Miss → Postgres → fill Redis with TTL.

Cache hit ratio targets > 95%. At 230k QPS, the DB only sees ~10k QPS — manageable.

### Write Path

```
POST /shorten
  ├── reserve code via local counter batch (or random + insert)
  ├── INSERT INTO urls ON CONFLICT do nothing
  ├── publish event (optional)
  └── return short URL
```

Write QPS is low (2.3k); single Postgres comfortably handles it. No sharding needed yet.

### Sharding (when growth demands)

When Postgres becomes the bottleneck (~10-20× current scale):

- Shard by **code hash** → keys evenly distributed. Range queries irrelevant for this domain.
- Replication factor 3.
- Vitess for MySQL, Citus for Postgres, or split application-side.

### Analytics

Click events are append-only, high-volume, never-updated. Wrong DB to put in Postgres — use ClickHouse:

```
on redirect:
  ├── return 301 immediately
  └── async send event to Kafka
                    │
                    ▼
            ClickHouse: clicks (code, ts, ip, ua)
```

ClickHouse stores billions of rows cheaply; rolls up for the stats endpoint.

### Failure Modes & Handling

| Failure | Mitigation |
|---------|-----------|
| Postgres primary down | Standby replica + auto-failover (Patroni); brief read-only window |
| Redis down | Fallback to DB; brief latency spike but no errors |
| One PoP down | CDN reroutes around it |
| Code counter exhausts batch | Re-fetch next batch; instance briefly throttles writes |
| Spam shortenings of malicious URLs | Block list / safe-browsing API on insert |

### Security

- **Open redirect abuse**: `bit.ly/abc1234` → `evil.com`. Mitigate via:
  - Phishing-host block lists.
  - Safe Browsing API integration.
  - Throttle per-source-IP shortenings.
  - Manual review of high-fanout shorteners.
- **Custom aliases**: rate-limit per user; profanity / impersonation filter.

### Custom Aliases

```
POST /shorten { custom_alias: "summer-sale" }
  → INSERT ... ON CONFLICT (code) DO NOTHING
  → 409 if taken
```

Reserve namespace: any custom alias must be > 4 chars to avoid collision with generated 7-char codes; or use a separate prefix (`/c/summer-sale`).

### Expiration & Deletion

Cron job purges expired rows. Or set TTL in Redis; lazy delete in DB on read.

GDPR: when user deletes account, hard-delete their `urls` rows (audit-log the deletion).

### Trade-offs

**301 vs 302**:
- 301 = permanent; browsers cache aggressively → great for performance, bad for click tracking.
- 302 = temporary; every click hits the server → analytics work, latency slightly worse.

Production bit.ly used 301 with separate fingerprint for tracking. Modern: 302 + edge analytics.

### Extensions (if asked "what would you add?")

- Geo-routing: serve from nearest region (Cloudflare KV / Edge cache).
- Multi-tenant isolation: per-account rate limits, vanity domains.
- Branded URLs: customer's domain → mapping.
- A/B testing: short code resolves to one of N URLs by user bucket.
- QR codes on shorten response.

### Common Mistakes

| Mistake | Better |
|---------|--------|
| Hash of URL as code without dedup logic | Deal with hash collisions explicitly |
| Random with no collision check | Eventually breaks at scale |
| Storing clicks in same DB as URLs | Different access pattern; mix slows everyone |
| No CDN | Origin sees 230k QPS unnecessarily |
| Treating analytics as a feature, not a stream | Real-time pipeline matters at scale |

> [!NOTE]
> The depth in this interview is **scale + key generation**. A candidate who says "use bcrypt" or "hash collisions never happen" hasn't done the math.

### Interview Follow-ups

- *"Can you make redirect latency < 10ms?"* — Edge KV (Cloudflare Workers KV, AWS Lambda@Edge with DynamoDB). Resolve at the CDN; only fall back to origin on miss.
- *"How would you handle a single viral URL with 1M QPS?"* — CDN absorbs; if origin cache hit, even origin can sustain a few hundred QPS. Edge caching is mandatory.
- *"How would you migrate from random codes to sequential?"* — Coexist. Code length grows naturally; sequential codes start above the random space (e.g., 8 chars).
