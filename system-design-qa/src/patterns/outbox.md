# Q: Transactional Outbox — atomic DB write + event publish.

**Answer:**

You write to your DB and need to publish a message — to Kafka, SNS, another service. The naïve "write then publish" loses events when the process crashes between the two steps. The **outbox pattern** solves this with one DB transaction.

### The Failure Mode

```
order = insert_order()        # ✅ committed
publish("OrderPlaced", order) # ❌ process crashed
                              #    DB has order, no event sent

# OR

publish("OrderPlaced", order) # ✅ sent
order = insert_order()        # ❌ failed
                              #    event refers to non-existent order
```

Either order, dual writes can diverge. No clever try/catch fixes it — the two systems aren't transactional together.

### The Solution

Write the event to an **outbox table in the same DB** in the same transaction. A separate process drains the outbox and publishes:

```
BEGIN;
INSERT INTO orders   (id, ...)                                VALUES (?, ...);
INSERT INTO outbox   (id, aggregate_type, event_type, payload, created_at)
                     VALUES (?, 'Order', 'OrderPlaced', ?, now());
COMMIT;
```

Either both rows are present or neither is. Atomic.

A relay process polls outbox, publishes, marks as sent:

```
SELECT * FROM outbox WHERE published_at IS NULL ORDER BY id LIMIT 100;
publish to Kafka
UPDATE outbox SET published_at = now() WHERE id IN (...);
```

### Outbox Schema

```sql
CREATE TABLE outbox (
    id              BIGSERIAL PRIMARY KEY,
    aggregate_type  TEXT NOT NULL,
    aggregate_id    TEXT NOT NULL,
    event_type      TEXT NOT NULL,
    payload         JSONB NOT NULL,
    headers         JSONB,
    created_at      TIMESTAMPTZ DEFAULT now(),
    published_at    TIMESTAMPTZ
);

CREATE INDEX ON outbox (published_at, id) WHERE published_at IS NULL;
```

The partial index makes polling cheap.

### Two Relay Strategies

**1. Polling**

A worker reads unpublished rows, sends them, marks them sent.

```python
while True:
    rows = db.fetch("SELECT * FROM outbox WHERE published_at IS NULL ORDER BY id LIMIT 100")
    if not rows:
        sleep(0.1)
        continue
    for r in rows:
        kafka.send(r.event_type, r.payload)
    db.exec("UPDATE outbox SET published_at = now() WHERE id IN ...")
```

Simple. Adds latency (polling interval). Concurrent workers can race — use `FOR UPDATE SKIP LOCKED` to safely parallelize:

```sql
SELECT * FROM outbox
WHERE published_at IS NULL
ORDER BY id
LIMIT 100
FOR UPDATE SKIP LOCKED;
```

**2. Change Data Capture (CDC)**

Tail the database's replication log (WAL/binlog). Tools: **Debezium**, **Maxwell**, AWS DMS.

```
Postgres WAL
    │
    ▼
Debezium connector
    │
    ▼
Kafka topic "outbox.events"
    │
    ▼
Downstream consumers
```

Sub-second latency, no polling load on the DB. Operationally heavier (Debezium cluster, schema management).

CDC bonus: it can stream changes from *any* table, not just `outbox`. Some systems skip the outbox table and CDC the business tables directly — works if events can be derived from row changes alone.

### Idempotent Publish

Even with outbox, **at-least-once delivery** is the rule. The relay might:
- Send to Kafka but crash before marking sent → resends on retry.
- Mark sent but Kafka acked late → next worker picks up.

Consumers must dedup by `outbox.id` or a business event ID. Standard idempotency pattern.

### Ordering

Per `aggregate_id` ordering preserved if you partition outbox dispatch by it:

```
publish to topic "orders" with key = aggregate_id
                              → Kafka guarantees per-partition order
```

Across aggregates, no order guarantee. That's usually fine; design events to be commutative or carry timestamps.

### Cleanup

Outbox grows forever if you don't prune:

```sql
DELETE FROM outbox WHERE published_at < now() - INTERVAL '7 days';
```

Keep enough history to replay if something downstream crashes. 7–30 days typical.

For very high volume: partition outbox by day (Postgres declarative partitioning) and drop old partitions.

### Outbox vs Other Patterns

| Pattern | Description | Trade |
|---------|-------------|-------|
| Outbox + polling | Above | Simple; small polling latency |
| Outbox + CDC | Above with Debezium | Sub-second; heavy ops |
| Dual writes (NO) | Write DB, then publish | Loses events on crash |
| Event sourcing | Events ARE the source of truth | Bigger paradigm shift |
| Listen/notify | DB notifies consumer (Postgres LISTEN) | Best-effort; fragile |
| Transactional Kafka | Kafka tx + DB write coordinated (XA-ish) | Operational pain; rare |

### Outbox + Inbox

Mirror pattern on consumer side: store inbound events in an **inbox** table before processing, in the same transaction as the business write:

```sql
BEGIN;
INSERT INTO inbox (event_id, event_type) VALUES (?, ?);   -- unique constraint dedups
INSERT INTO orders (...);                                  -- business work
COMMIT;
```

If event is replayed: unique constraint violation on `event_id` → handler safely no-ops. Bullet-proof at-least-once → exactly-once-effect.

### Real-World Example

E-commerce checkout:

```
BEGIN;
  INSERT INTO orders (id, user_id, total, status='pending');
  INSERT INTO outbox (event_type='OrderCreated', payload={...});
COMMIT;

Debezium → Kafka → consumers:
  - payment-service:   charge card
  - inventory-service: reserve stock
  - email-service:     send confirmation
  - analytics-service: record event

Each consumer dedups by event_id (from outbox).
```

### Common Mistakes

| Mistake | Fix |
|---------|-----|
| Forgetting unique key on event_id at consumer | Duplicate processing on retry |
| Outbox in a different DB than business data | Defeats the atomicity; same DB is the point |
| Synchronous relay (write then await publish in same tx) | Slow + couples DB tx to broker availability |
| Multiple relay workers without `SKIP LOCKED` | Race condition; duplicate sends |
| Storing massive payloads in outbox | Bloats DB; store references or signed URLs |
| No retention policy | Table grows unbounded |

> [!NOTE]
> Outbox isn't optional for any system that writes to a DB and publishes events. The "dual write" alternative looks fine in dev and silently loses events in production.

### Interview Follow-ups

- *"Why not just use a transactional Kafka producer?"* — KIP-98 Kafka transactions span Kafka topics, not your DB. You'd need XA-style 2PC; not natively supported by most DBs.
- *"What about NoSQL?"* — Same idea works if your NoSQL supports per-document/transaction atomicity. DynamoDB Streams (CDC) + outbox attribute works.
- *"How would you ship events ordered globally?"* — You usually don't need that. Per-aggregate ordering via partition key is enough. If you really need it, you've designed yourself into a single-writer bottleneck.
