# Q: Change Data Capture (CDC) — what it is, when to use it.

**Answer:**

CDC streams **every row change** in a database as an event. Built originally for data warehousing; now the backbone of microservice integration, materialized views, audit, and search indexes.

### What It Looks Like

```
DB write:                            CDC event:
INSERT INTO orders (id, total)    →  {
  VALUES (123, 50)                    "op": "create",
                                       "table": "orders",
                                       "after": {"id": 123, "total": 50}
                                     }

UPDATE orders SET total=60         →  {
  WHERE id=123                        "op": "update",
                                       "before": {"id": 123, "total": 50},
                                       "after":  {"id": 123, "total": 60}
                                     }

DELETE FROM orders WHERE id=123    →  {
                                       "op": "delete",
                                       "before": {"id": 123, "total": 60}
                                     }
```

### How It Works

DBs maintain a **transaction log** (Postgres WAL, MySQL binlog, MongoDB oplog) for replication. CDC tools tail that log:

```
DB ──writes──► Transaction Log ──tail──► CDC tool ──publish──► Kafka
```

Tool: **Debezium** (most common), **Maxwell** (MySQL), AWS DMS, GCP Datastream.

### Why Tail the Log (Not Poll the Tables)

- **Captures every change**, even ones rolled back via undo.
- **No load on application queries** (log is decoupled).
- **Captures ordering and transactions**.
- **Sub-second latency**.

Polling `SELECT * WHERE updated_at > ?` is the alternative; it misses deleted rows and adds DB load.

### Use Cases

| Use case | Why |
|----------|-----|
| Sync DB → search index (Elasticsearch) | Keep search fresh without app changes |
| Sync DB → analytics DB (ClickHouse, Snowflake) | Lower-latency than nightly batch |
| Cache invalidation | Bust Redis when underlying row changes |
| Microservice integration | Service B reacts to data changes in service A's DB |
| Audit trail | Every change is an event |
| Outbox alternative | CDC business tables directly |
| Cross-region replication | Stream changes to remote region |

### Outbox vs CDC

**Outbox**:
- App writes events to an outbox table inside the business transaction.
- CDC (or a poller) streams the outbox.
- App controls event semantics ("OrderShipped" with rich payload).

**Direct CDC** of business tables:
- No outbox; CDC the `orders` table.
- Downstream sees raw rows; less business meaning.
- Schema changes (column adds/drops) directly affect downstream.

Most teams use outbox. Direct CDC is for systems where you control both ends and want zero app changes.

### Schema Evolution

Tables change. CDC events must too:

- Add column: new events include the new field; old events don't (downstream tolerant).
- Drop column: old events still have it; downstream tolerant.
- Rename: best avoided; treat as drop+add.

Schema registry (Avro / Protobuf) on the Kafka side helps.

### Initial Snapshot + Streaming

When you turn on CDC, you usually want both:
1. **Snapshot**: existing data (initial load).
2. **Stream**: future changes from the WAL.

Debezium does this automatically: takes a consistent snapshot using `SELECT ... AS OF SCN` (Oracle) or `START TRANSACTION REPEATABLE READ` (Postgres), then transitions to streaming from the recorded LSN.

### Ordering

Per-table or per-key ordering preserved (CDC events flow through Kafka with key = primary key).

Across tables, ordering by transaction is best-effort. Multi-row transactions are visible as multiple events; downstream must reassemble if needed.

### Consumer Considerations

CDC events are **at-least-once**. Consumers must dedup:

- Use Kafka offset + topic-partition for resume.
- Use event's unique LSN / position field for dedup.

For "rebuild downstream from scratch": consume from offset 0; idempotent application of events makes this safe.

### Performance Impact on Source DB

CDC reads the WAL — but the WAL is already written for replication. Marginal extra load.

Caveats:
- Postgres: requires `wal_level = logical` and replication slots; abandoned slot fills disk.
- MySQL: row-based binlog format required.
- High write volume: CDC can produce huge Kafka traffic.

### Schema Tools

Debezium emits events in a specific Avro / JSON Schema format. Schema Registry mandatory at scale.

```json
{
  "schema": { "type": "record", ... },
  "payload": {
    "op": "u",
    "before": { ... },
    "after": { ... },
    "source": { "ts_ms": ..., "lsn": ... },
    "ts_ms": ...
  }
}
```

### Failure Modes

| Failure | Handling |
|---------|---------|
| Debezium connector crashes | Resumes from last offset in Kafka |
| Source DB master failover | New master, new WAL — connector reconnects |
| Old WAL purged before Debezium consumed | Lost events; alert on lag |
| Schema change downstream | Consumer must be tolerant (default values, alias) |

### Common Mistakes

| Mistake | Fix |
|---------|-----|
| No retention on WAL during downtime | Replication slot fills disk |
| Dual-writing to DB + Kafka (without outbox/CDC) | Lost events on crash |
| Treating CDC events as commands | They're after-the-fact; commands are intents |
| Ignoring ordering across tables | Stitch tx_id if cross-table consistency needed |
| Slow consumer | Lag balloons; back-pressures the source via replication slot bloat |

> [!NOTE]
> CDC is the canonical way to bridge ACID systems and event-driven downstream consumers. It's a different programming model — events instead of polled snapshots — but the payoff is massive: real-time integration with zero app code.

### Interview Follow-ups

- *"How would you migrate from a monolith DB to microservice DBs?"* — Stand up microservice; CDC the monolith table; populate new DB; cut over; remove the old.
- *"How does Debezium handle DDL?"* — Captures DDL events too (where supported); downstream may need re-config.
- *"How is CDC different from a trigger?"* — Triggers are synchronous, in-DB, affect write latency. CDC reads the log post-commit — async, no app impact.
