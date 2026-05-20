# Q: Design an ad-click aggregation pipeline.

**Answer:**

High-throughput event ingestion + aggregation + fraud detection. Tests **streaming**, **deduplication**, **windowing**, and **OLAP**.

### Requirements

**Functional**:
- Capture every ad click event.
- Aggregate per-ad clicks every minute / hour / day.
- Detect fraud / bot clicks.
- Report to advertisers in near-real-time.
- Bill advertisers accurately.

**Non-functional**:
- 1M clicks/sec peak.
- < 1 minute aggregation lag.
- Exact billing (no double-counting, no missed clicks).
- Replayability (recompute aggregations from events).

### Architecture

```
ad servers / SDK
       │
       ▼
   Event Collector  (stateless, geo-distributed)
       │
       ▼
   Kafka            (partitioned by ad_id or campaign_id)
       │
       ├──► Fraud Detection (stream)
       │         │
       │         ▼
       │   blocklist / Redis
       │
       ▼
   Aggregator (Flink / Spark Streaming)
       │
       ▼
   OLAP DB (ClickHouse / Druid)
       │
       ▼
   Reporting API + Billing
```

### Event Collection

Edge collectors (close to user) ingest clicks:

```
GET /click?ad_id=A1&user_id=U2&placement=P3&ts=&signature=...
```

Properties:
- Signed by ad server to prevent forgery.
- Return immediately (1×1 transparent pixel or redirect).
- Buffer locally; flush to Kafka in batches.

Throughput: ~1M events/sec. Stateless. Sized by traffic.

### Kafka

Partition by `ad_id` (or `campaign_id`):
- Ordering within an ad.
- Parallel consumers per partition.

Retention: 7 days for replay; longer in archive (S3).

### Deduplication

Same click might be reported twice (network retry, browser instability). Each event has unique `event_id`:

```
seen_events (Redis SET with TTL)
  on event:
    if SADD seen:event_id (returns 0): skip
    else: process
```

For exactness: dedup by `event_id` in the aggregator's state (Flink's keyed state).

### Fraud Detection

Real-time streaming filters:

- **Velocity**: clicks per IP per minute > N → mark suspicious.
- **Bot patterns**: identical user-agent, missing referer, headless browser detection.
- **Honeypot**: trap clicks (invisible to humans) that bots hit.
- **Repeat clicks**: same user_id clicking same ad more than once in T → discount.

ML model scores each click; rule-based blocks obvious bots; flagged events excluded from billing.

### Aggregation

Stream processor windowed per minute:

```
keyBy (ad_id, minute_window)
   .sum(clicks)
   .filter(not fraudulent)
   .writeTo(ClickHouse)
```

Flink / Spark Streaming / Kafka Streams handle this. State stored locally, checkpointed to durable store.

### Storage for Aggregations

**ClickHouse** (or Druid, Pinot):
- Append-only fact table.
- Columnar, compressed, fast aggregations.
- Stores per-minute counts; reports roll-up to hour/day at query time.

```sql
CREATE TABLE ad_clicks (
    ad_id        UUID,
    minute       DateTime,
    clicks       UInt32,
    impressions  UInt32,
    spend        Decimal(18, 4)
) ENGINE = SummingMergeTree()
ORDER BY (ad_id, minute);
```

### Billing Accuracy

Two-step process:

1. **Real-time aggregation** for dashboards (approximate, near-real-time).
2. **End-of-day reconciliation** (exact):
   - Re-process the day's events from Kafka (or S3 archive).
   - Apply final fraud rules.
   - Produce billable amounts.
   - Compare to real-time; flag discrepancies.

Advertisers billed on the reconciled number, not the live dashboard.

### Exactly-Once Semantics

Required for billing.

- Producer: Kafka idempotent + transactional → no duplicates in Kafka.
- Consumer (Flink): two-phase commit between Kafka offset commit and ClickHouse write.
- Or: append-only with `event_id` unique constraint at the OLAP layer.

In practice, *exactly-once effect* is achieved by at-least-once + dedup at the aggregator.

### Multi-Region

- Each region has local collectors + Kafka cluster.
- Cross-region replication (MirrorMaker) merges to a global topic for the aggregator OR each region aggregates locally and the central reporting sums up regions.

Trade-off: local aggregation = lower latency; central = simpler dedup.

### Data Retention

| Tier | Retention |
|------|-----------|
| Raw events in Kafka | 7 days |
| Raw events in S3 (archive) | 2 years (compliance) |
| Per-minute aggregates | 90 days |
| Per-hour aggregates | 1 year |
| Per-day aggregates | Forever |

Older granularities rolled up; raw events purged after retention.

### Replay

Bug in aggregator? Replay from Kafka:

```
1. Disable live aggregator.
2. Replay window from offset X.
3. Rewrite aggregations for the period.
4. Resume live.
```

Idempotent writes to ClickHouse (or full-replace partition) make this safe.

### API for Reporting

```
GET /reports?ad_id=A1&from=...&to=...&granularity=hour
```

Backed by ClickHouse aggregate queries. Cached for popular dashboards.

### Failure Modes

| Failure | Handling |
|---------|---------|
| Collector dies | LB drains; replicas absorb; tiny event loss possible |
| Kafka partition lost | Replication factor 3 — survives 1 broker loss |
| Aggregator crash | Resume from last checkpoint |
| ClickHouse shard down | Replicated; read from replica |
| Bad fraud rule false-positives many real clicks | Roll back rule; replay; refund advertisers |

### Common Mistakes

| Mistake | Fix |
|---------|-----|
| Writing each click direct to OLAP DB | DB collapses; use stream |
| Synchronous fraud check in click path | Latency; do real-time async + post-hoc detection |
| One global counter incremented per click | Bottleneck; shard or aggregate |
| Billing from live dashboard | Live is approximate; bill from reconciled batch |
| No replay capability | Lost when bug deployed |

> [!NOTE]
> Two systems coexist: a **fast loop** for the dashboard and a **slow loop** for billing. The fast loop accepts small inaccuracy for latency; the slow loop accepts latency for exactness. Never run billing off the fast loop.

### Interview Follow-ups

- *"How would you handle a viral ad with 100k clicks/sec on one ad?"* — Salt the key (split partition by `ad_id + bucket_n`); aggregate sub-keys then sum.
- *"How do you detect distributed bot networks?"* — Fingerprint patterns across IPs; ML over device + behavior + temporal features.
- *"How does this differ from page-view counting?"* — Same architecture; ad clicks need stricter exactness (money), heavier fraud filtering, and a billing reconciliation pass.
