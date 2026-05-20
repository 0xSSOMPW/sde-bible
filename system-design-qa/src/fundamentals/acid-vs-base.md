# Q: ACID vs BASE — what guarantees, and when to pick which?

**Answer:**

**ACID** is the transactional guarantee of relational databases: writes are all-or-nothing, isolated, and durable. **BASE** is the loose guarantee of NoSQL and distributed systems: writes eventually settle. Picking between them per-data-class is one of the most consequential design decisions in any system.

### ACID

- **Atomicity**: a transaction succeeds entirely or fails entirely. No partial commits.
- **Consistency**: a transaction leaves the database in a valid state (constraints, FKs, triggers).
- **Isolation**: concurrent transactions don't see each other's intermediate state.
- **Durability**: once committed, the data survives crashes (fsync to disk or replicated WAL).

Example: bank transfer.

```sql
BEGIN;
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
UPDATE accounts SET balance = balance + 100 WHERE id = 2;
COMMIT;
```

Either both rows change, or neither. No middle state where money disappears.

### Isolation Levels

The "I" in ACID is not binary — SQL standard defines four levels:

| Level | Prevents | Allows |
|-------|----------|--------|
| Read Uncommitted | nothing | dirty reads (rare in practice) |
| Read Committed (PG default) | dirty reads | non-repeatable reads, phantoms |
| Repeatable Read (MySQL InnoDB default) | non-repeatable reads | phantoms (PG: also prevents phantoms via snapshot) |
| Serializable | everything | — |

Anomalies:
- **Dirty read**: read uncommitted data from another tx.
- **Non-repeatable read**: same query, two values in one tx.
- **Phantom read**: same range query, different row count.
- **Lost update**: two tx read, both write, second overwrites first.
- **Write skew**: two tx read-then-write disjoint rows under a constraint that should hold globally.

`SERIALIZABLE` prevents all but is the slowest. Most apps run `READ_COMMITTED` and add `SELECT ... FOR UPDATE` or version columns to guard hot rows.

### BASE

- **Basically Available**: system responds (maybe with stale data).
- **Soft state**: state may change over time without input (replicas converge).
- **Eventual consistency**: given no new writes, all replicas converge.

Trade: relaxed consistency in exchange for:
- **Higher availability** under partitions.
- **Lower latency** (no synchronous cross-region writes).
- **Horizontal scale** beyond what 2PC allows.

Example: a social-network "like count." Showing 1,234 when it's truly 1,235 is fine.

### How to Decide

Per data class, ask:

| Question | ACID | BASE |
|----------|------|------|
| Are writes financial / legal / regulated? | ✓ | ✗ |
| Can a user tolerate stale reads for seconds? | ✗ | ✓ |
| Is the dataset > 1 TB and growing? | scaling pain | ✓ |
| Do you need multi-region writes < 100 ms? | hard | natural fit |
| Is referential integrity critical? | ✓ | ✗ |
| Does a stale value cause data loss or only UX glitch? | ACID if loss | BASE if UX |

In one system you almost always have **both**:

- ACID for orders, payments, inventory, user identity.
- BASE for feeds, recommendations, analytics counters, search indexes.

### Mixing Them Safely

Common pattern: write to ACID system of record, asynchronously propagate to BASE views.

```
Postgres (orders, ACID)
   │
   ▼ CDC (Debezium / outbox)
   ▼
Kafka
   │
   ▼
   ├──► ElasticSearch (search index)
   ├──► Redis (cache)
   └──► ClickHouse (analytics)
```

Source of truth is ACID. Derived views are BASE. Reconcile drift via periodic full-sync jobs or bookkeeping.

### Quasi-ACID at Scale

- **Spanner / CockroachDB**: distributed ACID. External consistency via TrueTime / hybrid logical clocks. Slower than single-node PG, faster than 2PC.
- **DynamoDB Transactions**: ACID across up to 100 items. Adds latency.
- **MongoDB multi-document transactions**: ACID within a shard; cross-shard adds 2PC.

These narrow the gap but don't eliminate it. Cost (latency, throughput) is real.

### Common Mistakes

| Mistake | Reality |
|---------|---------|
| "We use Postgres, so we're ACID" | Only inside transactions. Async fanouts to caches/search are eventual |
| "Eventual consistency = unreliable" | Means "consistent in the limit"; with bounded staleness it's measurable and engineerable |
| "Use NoSQL because it scales" | Most workloads fit on one well-tuned Postgres until very large |
| Running everything Serializable | Conflict aborts → retry storms; rarely the right default |
| Cross-microservice 2PC | Operational nightmare; use Saga + Outbox |

> [!NOTE]
> ACID is a feature you pay for. BASE is a property you build *around*. The biggest engineering wins come from segmenting your data correctly: ACID where wrongness costs money, BASE everywhere else.

### Interview Follow-ups

- *"How would you maintain a global like-count without write contention?"* — Sharded counter; per-shard atomic increment; periodic aggregation. BASE; final number eventually consistent.
- *"How do you keep an ACID DB and a search index in sync?"* — Outbox table in the same transaction; CDC stream → indexer. Avoid dual writes.
- *"What does `SELECT FOR UPDATE SKIP LOCKED` solve?"* — Lets multiple workers pull from a job table without blocking each other or pulling the same row.
