# Q: CQRS — Command-Query Responsibility Segregation.

**Answer:**

CQRS splits the **write model** (commands) from the **read model** (queries). Each is optimized for its access pattern, often using different schemas, stores, and even consistency levels. It pairs naturally with event-driven systems and is the easiest path to scaling read-heavy workloads.

### Without CQRS

One model, one schema, one DB:

```
        Service
   │             │
   write API     read API
       │           │
       ▼           ▼
   ┌─────────────────┐
   │   one table     │   <- shape compromises both
   └─────────────────┘
```

Writes need normalized rows; reads need denormalized, pre-joined data. The single schema is a compromise that satisfies neither.

### With CQRS

Two models:

```
   Commands              Queries
       │                    │
       ▼                    ▼
   Write side          Read side
   (source of truth)   (denormalized views)
       │                    ▲
       │  events / CDC      │
       └────────────────────┘
```

- Write side: small, normalized, transactional.
- Read side: denormalized, pre-joined, query-shaped.
- Synced asynchronously via events or CDC.

### Concrete Example

E-commerce order tracking.

**Write side** (Postgres):

```sql
orders   (id, user_id, status, ...)
items    (order_id, sku, qty, price)
```

Strict ACID, used by `POST /orders`, `PATCH /orders/:id`.

**Read side** (Elasticsearch + Redis):

```json
// One denormalized doc per order, indexed by user
{
  "order_id": "o123",
  "user_id":  "u42",
  "status":   "shipped",
  "items":    [{ "sku": "abc", "name": "Widget", "qty": 2, "subtotal": 19.98 }],
  "total":    19.98,
  "shipped_at": "2025-...",
  "tracking_url": "..."
}
```

Used by `GET /users/:id/orders` and `GET /orders/:id`. One query returns everything the UI needs.

Sync: on every write, publish events; a projection updates Elasticsearch.

### Where Each Side Lives

| Side | Storage choice | Reason |
|------|----------------|--------|
| Write | Postgres / RDBMS | ACID, constraints, transactions |
| Read | Elasticsearch | Full-text + filters |
| Read | Redis | Sub-millisecond reads |
| Read | ClickHouse / Druid | Analytics aggregates |
| Read | Materialized views in same DB | Smallest leap from non-CQRS |

You can also stay in one DB and use materialized views as the read model — lightweight CQRS without polyglot.

### When CQRS Is Right

- Read patterns very different from write (e.g., complex aggregations).
- Reads >> writes (10:1, 100:1).
- Need multiple read shapes (dashboard, mobile, admin) without each pounding the write DB.
- Already event-driven.

### When CQRS Is Wrong

- CRUD apps with simple, symmetric read/write.
- Strong read-your-writes required everywhere (CQRS's read side lags).
- Team can't afford the operational complexity.

CQRS adds **eventual consistency** to a path that was previously instant. Make sure the UX can absorb it.

### Read-Your-Writes Challenges

After `POST /orders`, the user immediately refreshes. The read store hasn't been updated yet → user sees no order.

Mitigations:
1. **Optimistic UI**: client renders the just-written value locally.
2. **Read-through cache**: write side updates a fast cache synchronously, slow read store async.
3. **Version pinning**: client sends `min_version=N`; server waits or routes to write DB until projection catches up.
4. **Hybrid**: critical reads from write DB, list/search reads from projection.

### Eventual Consistency Story

Write happens; projection updates after a delay (50ms – several seconds typically). Measure:

```
projection_lag = now() - latest_processed_event_timestamp
```

Alert at thresholds; show "syncing" UI; gate auditable actions on linearizable reads.

### CQRS Without Event Sourcing

CQRS is often confused with Event Sourcing. They're separate:

- **CQRS**: separate read/write models.
- **Event Sourcing**: log of events is the source of truth (write side stores events, not state).

You can do CQRS with a traditional write side (just publish events for the projection). See [Event Sourcing](./event-sourcing.md).

### Projection Patterns

A **projection** is the code that builds the read model from events.

```
on OrderPlaced(id, user_id, items):
    upsert read_orders (id, user_id, items, status='placed')

on OrderShipped(id, tracking):
    update read_orders set status='shipped', tracking_url=... where id=...

on OrderCancelled(id):
    update read_orders set status='cancelled' where id=...
```

Properties:
- Idempotent (replays don't break).
- Eventually consistent.
- Re-buildable: nuke the read store, replay all events from offset 0 → rebuild from scratch.

That last property is gold. Bug in projection logic? Fix code, rebuild from event log. Want a new read shape? Add a projection, replay history.

### Side Reads on Write Side

A common variant: simple/critical reads (`GET /orders/:id` right after `POST`) hit the write side; expensive/list reads (`GET /users/:id/orders?status=...`) hit the projection.

```
                    ┌── write tier (Postgres)
GET /orders/:id ────┤  read-after-write reads
                    │  occasional simple reads
POST /orders ───────┘

GET /search ────────► read tier (Elastic)
```

Best of both: tight consistency where it matters; scale where the queries are.

### Common Mistakes

| Mistake | Fix |
|---------|-----|
| CQRS for a simple CRUD app | Overengineering — single DB is fine |
| Sync writes to both stores | Defeats the point; use events/CDC |
| No projection lag monitoring | Stale data invisible until user complaints |
| Projection that can't rebuild | Don't lose the ability to nuke and replay |
| Mixing read schema migrations with write schema | Keep schemas independently versionable |

> [!NOTE]
> CQRS shines when read and write demands are genuinely different. If they're the same, you're paying complexity for nothing. Use it where it pays back.

### Interview Follow-ups

- *"How is CQRS different from a read replica?"* — Replica has the same schema. CQRS allows different schema, different DB engine, different consistency.
- *"How do you keep projections in sync with new code?"* — Version them; deploy in parallel; flip traffic when caught up; old projection deleted after burn-in.
- *"What's the projection failure recovery?"* — Resume from last committed offset; idempotent operations → safe replay.
