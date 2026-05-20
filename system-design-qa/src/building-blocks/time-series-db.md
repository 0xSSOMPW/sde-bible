# Q: Time-series databases — what makes them different?

**Answer:**

A time-series DB (TSDB) is optimized for append-only, timestamp-indexed data with high write volume and range-scan reads. Trying to do this in a normal RDBMS is a path to pain.

### What Makes Data "Time-Series"

- Each record has a timestamp.
- New data is mostly appended at "now."
- Updates are rare; deletes are bulk (retention).
- Queries scan ranges by time: "last 5 min," "yesterday."
- Aggregations dominate: rate, sum, p99 over windows.

Use cases: monitoring metrics, IoT sensor data, financial ticks, user activity events.

### Popular TSDBs

| TSDB | Lineage |
|------|--------|
| InfluxDB | Native TSDB; popular for IoT |
| Prometheus | Pull-based monitoring |
| TimescaleDB | Postgres extension; SQL+TSDB |
| Druid / Pinot | Real-time analytics columnar |
| ClickHouse | Columnar OLAP, doubles as TSDB |
| OpenTSDB / KairosDB | HBase-backed |
| VictoriaMetrics | Prometheus-compatible, denser |

### Optimizations

**Append-only writes**: no random writes; sequential disk patterns; high throughput.

**Compression**: Gorilla algo (timestamps as delta-of-delta, values as XOR diff) → ~1.4 bytes/sample.

**Time-partitioned storage**: each chunk covers a time window; old chunks dropped wholesale.

**Indexes by (series, timestamp)**: range queries find the chunk + scan.

**Downsampling**: roll up raw → 1-min → 5-min → 1-hour at increasing retention.

### Schema: Series

```
series_id = hash(metric_name + label_set)
data = [(ts, value), (ts, value), ...]
```

A series is a unique combination of labels. Each new label combination creates a new series.

**Cardinality** = total unique series. Bad labels (user_id, request_id) explode cardinality → OOM.

### Querying

Prometheus PromQL example:

```
sum(rate(http_requests_total{status=~"5.."}[5m])) by (service)
```

InfluxDB Flux:

```
from(bucket: "metrics")
  |> range(start: -1h)
  |> filter(fn: (r) => r._measurement == "cpu" and r.host == "h1")
  |> aggregateWindow(every: 1m, fn: mean)
```

TimescaleDB SQL:

```sql
SELECT time_bucket('1 minute', time) AS bucket, avg(value)
FROM cpu_usage
WHERE host = 'h1' AND time > now() - INTERVAL '1 hour'
GROUP BY bucket
ORDER BY bucket;
```

### Retention

Time-based retention is built in:

```
keep raw for 7 days
keep 1-min downsampled for 30 days
keep 1-hour downsampled for 1 year
```

Dropping a time window = dropping a chunk file. Trivial.

### vs Generic RDBMS

If you try Postgres for metrics at scale:
- 100k inserts/sec hits write throughput.
- Indexes bloat from constant inserts.
- Vacuum overhead.
- Time-range queries do sequential scan on huge tables.

TimescaleDB sidesteps this by **chunking** (auto-partitioning by time) on top of Postgres + columnar compression. Best of both: SQL + TSDB performance.

### vs OLAP (ClickHouse, BigQuery)

OLAP DBs are great for time-series too. ClickHouse, Druid, Pinot:
- Columnar storage.
- Excellent aggregations.
- Higher query latency than dedicated TSDBs but more flexible (joins, distinct counts).

For metrics: TSDB. For events with rich querying: OLAP. The line blurs.

### Cardinality Management

Sources of explosion:
- Per-user labels.
- Per-request IDs.
- Per-pod UUIDs in K8s.
- Free-form text labels.

Rule of thumb:
- Labels: < 10 values per label, < 10 labels.
- Cardinality budget per metric: ~10,000 series.
- High-cardinality goes to logs/events, not metrics.

### Scale

```
Million-series TSDBs commonplace.
Tens-of-millions with sharded/clustered solutions.
Hundreds-of-millions: managed (Mimir, VictoriaMetrics cluster, M3DB).
```

Single node holds ~1M series, 100k samples/sec.

### Multi-Tenant / Federated

For organization-wide metrics:
- Per-team / per-cluster TSDB.
- Federation: central queries fan out to clusters.
- Or: managed multi-tenant (Cortex, Mimir, Datadog).

### Common Mistakes

| Mistake | Fix |
|---------|-----|
| Using RDBMS for high-volume time-series | TSDB or OLAP |
| User ID as a label | Cardinality death |
| Querying months at raw resolution | Use downsampled |
| Single global retention | Per-metric retention sized to value |
| Logging individual events as metrics | Use logs/traces; metrics are aggregates |

> [!NOTE]
> Time-series databases trade flexibility for compression + write performance. They're not "a SQL database with a timestamp column" — they're a specialized engine and need to be treated as such.

### Interview Follow-ups

- *"When would you NOT use a TSDB?"* — Ad-hoc analytics with complex joins; rich relational queries — use a warehouse instead.
- *"How would you store both metrics and traces?"* — Metrics in TSDB; traces in dedicated trace store (Tempo, Jaeger); logs in Loki/Elastic. Three pillars, three stores.
- *"What's downsampling, and what does it cost?"* — Aggregate raw points into coarser buckets (1m → 5m → 1h) over time. Cost: small CPU to compute; you lose precision but save 12–60× storage.
