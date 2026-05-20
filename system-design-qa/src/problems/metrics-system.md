# Q: Design a metrics / monitoring system (Prometheus / Datadog-scale).

**Answer:**

Ingest and query high-cardinality time series at scale. Tests **TSDB internals**, **labels / cardinality**, **downsampling**, **query routing**, and **alerting**.

### Requirements

**Functional**:
- Ingest metrics from millions of agents.
- Query (rate, sum, p99) over arbitrary time ranges.
- Alert when metrics breach thresholds.
- Dashboards (Grafana-like).
- Retention: 15 days hot, 1 year cold.

**Non-functional**:
- 10M+ unique series.
- 10M data points/sec ingest.
- p99 query < 1 s for hot data.
- High availability.

### What a Time Series Is

```
metric_name { label_k1=v1, label_k2=v2, ... }  ts=12345  value=42
```

A `series` = unique combination of (metric_name, labels). One series produces many (timestamp, value) samples over time.

### Storage Model

Each series is a sequence of (ts, value) pairs. Optimized writes are **append-only**.

```
Block layout (typical TSDB):
  index:   series_id → list of (ts_block, offset)
  chunks:  ts_block → compressed (ts, val) pairs
```

Compression: Gorilla algorithm (delta-of-delta on timestamps, XOR on float values) → ~1.4 bytes/sample average.

10M samples/sec × 1.4 B × 86400 s ≈ 1.2 TB/day raw.

### Cardinality

Each unique label combination is a series. Bad labels explode cardinality:

```
{ user_id=42 }    → 100M users = 100M series   ← death
{ status=200 }    → small cardinality          ← good
```

A series has fixed metadata cost; explosion = OOM.

Rule: labels are *categorical, low-cardinality*. Anything user-bound, request-bound, time-bound → not a label.

### Architecture

```
agents (every host / pod)
     │  push or scrape
     ▼
Ingest nodes  (load-balanced, stateless)
     │
     ├─► write to local TSDB shard
     │
     ▼
Replication (Raft / quorum)
     │
     ▼
Long-term storage  (S3 / object store with index)
     │
     ▼
Query layer  (parallel fanout)
     │
     ▼
Alertmanager / Dashboard
```

### Push vs Scrape

- **Scrape (Prometheus)**: monitoring server polls each target's `/metrics` HTTP endpoint.
  - Pros: explicit service discovery; aliveness from scrape success.
  - Cons: target must be reachable from monitor; firewall complications.
- **Push (Statsd, Datadog, OpenTelemetry)**: agent on host pushes to ingest.
  - Pros: agents behind NAT, easy multi-tenant.
  - Cons: harder to detect "missing host."

Modern systems support both via OpenTelemetry collector.

### Ingest Path

```
ingestion API receives batch
  → validate (cardinality, schema)
  → write to in-memory + WAL
  → ack
```

Backpressure: reject when shard near capacity (return 429).

Sharding: hash by series ID → ingest shard. Each shard owns ~1/N series.

### Replication

For each shard:
- 3-node Raft cluster.
- Write to majority before ack.
- Read from any (eventual; reads from leader for strong).

### Long-Term Storage

Hot data: local SSD on ingest nodes (last 24–48 h).
Warm: queryable object storage (e.g., Thanos, Cortex, Mimir → S3).
Cold: archived; queryable with higher latency.

Downsampling: aggregate to 1-min, 5-min, 1-hour buckets for long retention.

```
hot      (raw 15s):       2 days
medium   (1 min):        30 days
cold     (5 min):       180 days
deep     (1 hour):       2 years
```

### Query Path

PromQL example:

```
sum(rate(http_requests_total{status=~"5.."}[5m])) by (service)
```

Query engine:
1. Parse PromQL.
2. Identify which series match labels.
3. Fetch samples for each series.
4. Apply functions (rate, sum, histogram_quantile).
5. Group by labels.

For long ranges, route to downsampled tier.

### Alerting

Alerts evaluated periodically (every 30s/1min):

```
expr: rate(errors[5m]) > 100
for:  5m              # threshold must hold for 5 minutes
```

Firing alert → Alertmanager → routing → notification (PagerDuty, Slack).

Suppression: alert dependencies, maintenance windows, deduplication of identical alerts.

### Service Discovery

- Kubernetes API: list pods + labels → scrape targets.
- Cloud provider APIs: EC2 instances, GCE VMs.
- Consul, etcd, DNS-based.

### Scaling

- **Single Prometheus**: ~1M series, ~100k samples/sec — fine for medium clusters.
- **Federated Prometheus**: shard per cluster, aggregate via federation.
- **Cortex / Thanos / Mimir**: distributed, multi-tenant TSDB on S3.
- **Datadog / Grafana Cloud / NewRelic**: managed SaaS.

### Multi-Tenant Considerations

- Per-tenant rate limits + cardinality caps.
- Per-tenant retention.
- Quotas with billing implications.

### Failure Modes

| Failure | Handling |
|---------|---------|
| Ingest node crashes | Targets retry; data buffered briefly at agent |
| Sharding hot key | Hash uneven; rebalance shards |
| Query takes forever | Limit max series, max samples, max duration |
| Storage backend lag (S3) | Reads block; serve from hot tier if possible |

### Common Mistakes

| Mistake | Fix |
|---------|-----|
| Using `user_id` as a label | Cardinality explosion; use a span/log instead |
| Querying months at raw resolution | Use downsampled tier |
| No retention policy | Disk fills; queries slow |
| Single Prometheus for whole org | Hits limits; federate or use Cortex/Mimir |
| Alert evaluation every 1s | Massive load; 30s is usually fine |

> [!NOTE]
> The dominant constraint in metrics systems is **cardinality**, not data volume. Engineers add a "user_id" label once and bring the whole TSDB to its knees. Cardinality budgets are real; enforce them.

### Interview Follow-ups

- *"Difference between metrics, logs, traces?"* — Metrics: aggregate counters over time. Logs: discrete events. Traces: causally-linked spans across services. Different stores, different costs, different uses.
- *"How is OpenTelemetry related?"* — Vendor-neutral SDK + protocol for emitting all three; you choose the backend (Prometheus, Tempo, Loki, Datadog).
- *"What is histogram_quantile?"* — Computes percentiles from server-side histograms; client-side percentiles can't be aggregated, server-side ones can.
