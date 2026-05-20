# Q: Sharding & partitioning — strategies, key choice, rebalancing, gotchas.

**Answer:**

**Partitioning** is splitting a dataset into subsets. **Sharding** is partitioning across separate machines. You shard when one machine can't hold the working set, serve the QPS, or survive the disk-write load.

### The Three Strategies

| Strategy | How | Pros | Cons |
|----------|-----|------|------|
| Range partitioning | Rows grouped by key range | Range queries efficient | Skew → hot shards |
| Hash partitioning | `hash(key) % N` | Even distribution | No range queries |
| Directory / lookup | Explicit map (key → shard) | Flexible | Lookup table is itself bottleneck |
| Consistent hashing | Hash on a ring | Smooth rebalance | More moving parts |

### Range Partitioning

```
shard A: keys [aaa-bbb)
shard B: keys [bbb-ccc)
shard C: keys [ccc-zzz)
```

Used by HBase, BigTable, MongoDB (range mode), CockroachDB. Splits/merges shards automatically as data grows.

Strength: scan `WHERE key BETWEEN x AND y` reads one (or few) shards.
Weakness: monotonically increasing key (e.g., timestamp) → all writes to last shard.

### Hash Partitioning

```
shard = hash(user_id) % N
```

Spreads writes uniformly. Used by Cassandra, DynamoDB, MySQL (by app), Redis Cluster.

Strength: even load (if keys are well-distributed).
Weakness:
- No `WHERE key BETWEEN ...` range queries — scatter-gather across all shards.
- Adding shards changes `% N` → almost all keys move. (See: consistent hashing.)

### Consistent Hashing

Place shards and keys on a ring 0..2^64. Each key maps to the next shard clockwise. Adding/removing one shard moves only `1/N` of the keys.

```
       0
        ╲
         ●─── shard A (token 100)
       ╱
    keys 50..100  → A

    keys 100..220 → B
         ●─── shard B (token 220)
       ╱
   ...
```

Refinement: each shard owns multiple **virtual nodes** (tokens) on the ring → smoother rebalance. Cassandra uses 256 vnodes/server by default.

See [Consistent Hashing](../patterns/consistent-hashing.md) for the full mechanics.

### Choosing a Shard Key

The most consequential decision. Bad shard keys cause:
- **Hot shards** (one shard takes 90% of load).
- **Scatter-gather queries** (every read hits every shard).
- **Cross-shard transactions** (slow, complicated).

Good shard key properties:
- **High cardinality** (millions+ of distinct values).
- **Uniform access pattern** (no celebrity hot keys).
- **Co-locality**: rows accessed together share a shard.
- **Stable over time** (not monotonically increasing).

Common keys:

| Domain | Good shard key |
|--------|---------------|
| SaaS multi-tenant | `tenant_id` |
| Social network | `user_id` |
| E-commerce | `customer_id` (read-heavy) or `order_id` (write-heavy) |
| IoT | `device_id` (avoid timestamp alone) |
| Geo data | `geohash(lat, lng)` |

Bad keys:
- `created_at` (monotonic → hot shard).
- `country` (skewed; US dominates).
- `status` (low cardinality).
- Mutable fields (key change = data movement).

### Composite Shard Keys

Combine fields for cardinality + locality:

```
shard_key = hash(tenant_id) + range(created_at)
```

Hash spreads load across shards; range within a shard keeps a tenant's recent data together. Used by MongoDB's compound shard keys and DynamoDB's PK + SK design.

### Hot Shard Problems

Symptoms: one shard at CPU/disk limit while others idle.

Causes:
1. **Skewed key distribution** (one tenant huge).
2. **Sequential key** (timestamp, auto-increment ID).
3. **Hot row** within a shard (celebrity user).

Mitigations:
- **Salt the key**: prepend a random small bucket → `bucket_n_userid`. Decompose on read with scatter-gather.
- **Re-key by composite**: hash a more uniform field.
- **Split the hot shard** (range partitioning supports this automatically).
- **Tiered access**: separate read replicas just for the hot tenant.

### Rebalancing

Adding capacity needs to move some data without taking the system offline.

| Approach | Mechanism |
|----------|-----------|
| Stop the world, copy, restart | Smallest engineering effort, biggest outage |
| Resharding (range) | Split a shard's range in half; move half |
| Consistent hashing + vnodes | Add new node → it claims `1/N` of vnodes from existing nodes |
| Logical → physical mapping | App holds a directory of logical shards → physical hosts; rebalance by remapping |

Production systems: vnode + background streaming. Cassandra, ScyllaDB, DynamoDB do this transparently.

### Cross-Shard Queries

Queries that span multiple shards are slow:

```
SELECT * FROM orders WHERE customer_email = ? ; -- shard key is user_id
```

Options:
1. **Secondary index on email** → maintained per shard, queried via scatter-gather.
2. **Global secondary index** → separate index store keyed by email (writes are now cross-shard).
3. **Application-level fanout** → query all shards in parallel, aggregate.

For aggregate queries (counts, sums), use a separate analytics store (ClickHouse, Druid).

### Cross-Shard Transactions

Hard. Options ordered by complexity:

1. **Avoid them**. Co-locate via shard key design.
2. **Saga** — distributed multi-step workflow with compensation. See [Saga Pattern](../patterns/saga.md).
3. **2PC** — synchronous, blocking, complex. Last resort.
4. **Use a distributed SQL DB** (Spanner, CockroachDB, YugabyteDB) that does it for you.

### Schema Per Shard

Each shard typically has the **same schema**. Migrations must apply to all shards — usually orchestrated by a tool (Vitess, Citus) or careful application code.

Multi-tenant per-customer-DB pattern (one DB per tenant) trades shard-management complexity for migration complexity — each new schema change is N migrations.

### Examples by System

| System | Strategy |
|--------|---------|
| Cassandra | Consistent hash (Murmur3) + vnodes |
| DynamoDB | Hash on partition key; range on sort key within partition |
| MongoDB | Range or hash, configurable per collection |
| Postgres + Citus | Hash by `distribution_column` |
| Vitess (MySQL) | Hash on `vindex`, lookup-vindex for secondary |
| Kafka | Hash on producer key, partition count |
| Redis Cluster | CRC16 → 16384 hash slots |

### Common Mistakes

| Mistake | Result |
|--------|--------|
| Shard key = `created_at` | All new writes hit one shard |
| Shard early without need | Adds complexity for years |
| Skip secondary indexes | Cross-shard scatter-gather kills latency |
| Hot tenant on a multi-tenant shard | Cripples neighbors; need isolation/tiered DBs |
| No plan for resharding | Future you suffers; build directory layer early |
| Cross-shard `JOIN` | Use denormalization or a separate analytics store |

> [!NOTE]
> Sharding is a one-way door. Pick the key with care; getting it wrong costs months of migration. Better to design for shardability and *not* shard than to shard with the wrong key.

### Interview Follow-ups

- *"How would you migrate from one shard key to another?"* — Dual write to old + new (mapped by both keys); backfill old data; switch reads; remove old after burn-in.
- *"How do range queries work on hash-partitioned data?"* — They don't, efficiently. Use a separate index (search engine, materialized view) keyed appropriately.
- *"What's the failure mode of consistent hashing under node loss?"* — Node's range is taken over by its neighbor → load doubles. With vnodes, load spreads across many neighbors.
