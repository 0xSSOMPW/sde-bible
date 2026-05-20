# Q: SQL vs NoSQL — when to pick which?

**Answer:**

"SQL vs NoSQL" is a misleading split — there are at least five NoSQL families with very different properties. The right question is "given my access patterns and constraints, which data model and consistency story fits?"

### The Families

| Family | Examples | Best for |
|--------|----------|----------|
| Relational (SQL) | Postgres, MySQL, Oracle, SQL Server | ACID transactions, ad-hoc queries, normalized data |
| Document | MongoDB, DynamoDB, Couchbase | Schema flexibility, nested objects, per-document access |
| Wide-column | Cassandra, ScyllaDB, HBase, Bigtable | Massive write throughput, time-series, log-like |
| Key-value | Redis, Memcached, DynamoDB, RocksDB | Fast lookups by key, caching, session store |
| Graph | Neo4j, ArangoDB, JanusGraph, Neptune | Relationship traversal, recommendation, fraud |
| Search | Elasticsearch, OpenSearch, Meilisearch | Full-text + ranking, filtered search |
| Time-series | InfluxDB, Timescale, Prometheus | Metrics, IoT telemetry |

### The Right Mental Model

Don't pick one for the whole system. Polyglot is the norm:

```
Source of truth:    Postgres        (orders, users — ACID)
Hot reads:          Redis            (sessions, leaderboards)
Search:             Elasticsearch    (full-text)
Analytics:          ClickHouse       (rollups)
Object/blob:        S3                (uploads)
Event bus:          Kafka             (audit, fanout)
Time-series:        Timescale         (metrics)
```

Pick the **source of truth** carefully; derived stores can be rebuilt.

### When SQL Is Right

- Strong consistency required (financial, regulated, identity).
- Complex queries (joins, aggregates) you can't predict in advance.
- Mature tooling (migrations, backups, monitoring) matters.
- Data fits in one logical DB (under a few TB hot working set).
- You want referential integrity enforced.

Postgres is the boring, correct answer for ~80% of new services.

### When NoSQL Is Right

Each family for different reasons:

**Document store** (Mongo, Dynamo):
- Schema evolves rapidly.
- Object naturally hierarchical (a "product" with nested specs/reviews).
- Single-record reads dominate.
- Multi-tenancy with isolated schemas per tenant.

**Wide-column** (Cassandra):
- Write throughput >100k QPS that one RDBMS can't deliver.
- Time-series or log-like data partitioned by entity + time.
- Multi-region active-active with tunable consistency.

**Key-value** (Redis):
- Lookups by exact key.
- TTL-based eviction (sessions, rate-limits).
- Low-latency requirements (sub-millisecond).

**Graph DB**:
- Traversals more than 2–3 hops are common.
- "Friends of friends," fraud rings, recommendation graphs.
- (Often: Postgres + recursive CTE handles 2-hop just fine.)

**Search**:
- Full-text, faceted search.
- Ranking by relevance, not just filtering.

**Time-series**:
- Append-only metrics.
- Range queries on timestamp.
- Compression of similar adjacent values.

### Schema Flexibility

A common reason teams reach for NoSQL: "we don't know the schema yet."

Reality:
- Postgres supports `JSONB` columns with indexes — gives you schemaless with optional structure.
- "Schemaless" doesn't mean "no schema" — it means the schema is in app code, harder to audit.
- Real schema evolution still needs care.

Pattern: Postgres with JSONB for the flexible bits, typed columns for the well-defined bits. Avoid full document-DB unless you really need it.

### Access Pattern Matters More Than Tech

NoSQL forces you to **design for access patterns up front** (DynamoDB famously). SQL lets you write any query and add an index later.

Example: a "user's recent orders" page.

```
SQL (Postgres):
  SELECT * FROM orders WHERE user_id = ? ORDER BY created_at DESC LIMIT 20;
  index: (user_id, created_at DESC)

DynamoDB:
  PK = user_id, SK = created_at
  Query(PK = u123, SK desc, limit 20)
```

Same access pattern, both work. SQL gives you `GROUP BY status` for free; DynamoDB needs a second index.

### Joins

A common claim: "NoSQL doesn't do joins." Mostly true. The reply is denormalize:

- Embed related data in the document.
- Maintain a materialized view.
- Pre-compute the join offline.

If your model needs joins on every read, SQL is the lower-friction choice.

### Transactions

| Class | Multi-row transactions |
|-------|------------------------|
| SQL | Always (ACID) |
| DynamoDB | Up to 100 items, single region |
| MongoDB | Multi-document since 4.0 (replica-set scope) |
| Cassandra | Lightweight transactions (Paxos, slow) |
| Redis | MULTI/EXEC for batch, optimistic |

If your business logic crosses many entities atomically, prefer SQL.

### Scaling Story

| Family | Scale-up path |
|--------|--------------|
| SQL | Vertical scale → read replicas → application-side sharding (Vitess, Citus) |
| Distributed SQL (Spanner/Cockroach) | Horizontal natively |
| DynamoDB / Cassandra | Horizontal natively (partition key is the limit) |
| Redis Cluster | Hash slots across nodes |

"Scales horizontally out of the box" was the original NoSQL appeal. Modern SQL has caught up — distributed SQL is no longer rare.

### Cost & Operability

- **Managed SQL** (RDS, Aurora, Cloud SQL): expensive at scale, very mature ops.
- **Managed NoSQL** (DynamoDB): pay per request + storage; cheap at low scale, expensive at extreme scale.
- **Self-hosted**: cheaper compute but you own the on-call.

DynamoDB's "auto-scaling" is real but bills can surprise. Cassandra is brutal to run yourself.

### Common Mistakes

| Mistake | Better |
|---------|--------|
| MongoDB for relational data | Postgres + JSONB if flexibility matters |
| Cassandra for low-write workload | Pure overengineering |
| Redis as primary store | OK for some use cases, but no built-in durability beyond AOF |
| Choosing "modern" tech over team experience | Boring tech wins long-term |
| Single DB for OLTP + analytics | Hot/cold path competes; split via CDC |

### Decision Heuristics

- **Default**: Postgres for source of truth. Add specialized stores as patterns emerge.
- **Write QPS > 50k**: think Cassandra/DynamoDB or Postgres with Citus.
- **Search**: don't do `LIKE '%x%'` in Postgres at scale — use Elasticsearch.
- **Sub-millisecond reads**: Redis in front of the DB.
- **Multi-region active-active**: Cassandra, Cosmos, or Spanner/Cockroach.
- **Data > 100 TB hot**: distributed SQL or columnar warehouse.

> [!NOTE]
> The most common production stack is Postgres + Redis + S3 + a queue. That's enough for most consumer-scale apps. Reach for more exotic stores only when access patterns demand it.

### Interview Follow-ups

- *"You said Postgres. How would you scale it past one machine?"* — Read replicas; partition table by time/customer; eventually shard with Citus or Vitess; or migrate to distributed SQL.
- *"Why is DynamoDB schemaless?"* — Schema is enforced at app level; tables only require PK (and optional SK). Trades validation for flexibility.
- *"What's HTAP?"* — Hybrid Transactional/Analytical Processing — single DB serving OLTP + analytics. SingleStore, TiDB, Spanner. Niche; usually splits via CDC are simpler.
