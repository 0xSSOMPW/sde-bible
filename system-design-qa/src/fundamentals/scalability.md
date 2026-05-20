# Q: What is scalability, and what are the axes you can scale along?

**Answer:**

A scalable system handles more load by adding resources, ideally **proportionally** (2× resources → 2× capacity). The three classical axes — vertical, horizontal, and **functional** — describe different strategies. Real systems scale along all three at once.

### The Scale Cube

Coined by Martin Abbott, this is the canonical mental model:

```
                  Z-axis: data partition / shard
                       ▲
                       │
                       │
                       │
       Y-axis (functional split, services)
              ◄────────┼────────►
                       │
                       │
                       ▼
                  X-axis: clone / horizontal
```

- **X**: run more copies of the same thing behind a load balancer.
- **Y**: split by *function* — separate services for orders, payments, search.
- **Z**: split by *data* — shard customers by region, user_id mod N.

### X-Axis: Horizontal Scaling

Run N identical app servers behind a load balancer. Prerequisites:

- **Stateless** application tier (no in-memory session that survives across requests).
- Externalize state to a shared store (Redis, DB).
- Idempotent request handling (because retries may hit different instances).

```
        ┌──── App-1
LB ─────┼──── App-2     → DB / cache / queue
        └──── App-3
```

Cheap if you've designed for it. Almost free with K8s deployments + HPA.

Limits:
- Database eventually becomes the bottleneck.
- Shared mutable state (cache, queue) still has its own limits.
- Network coordination grows non-linearly past some point.

### Y-Axis: Functional Decomposition

Split the monolith into services by domain:

```
Monolith                    Microservices
┌──────────────┐            ┌─────────┐  ┌─────────┐
│ Orders       │            │ Orders  │  │ Payments│
│ Payments     │     →      └─────────┘  └─────────┘
│ Inventory    │            ┌─────────┐  ┌─────────┐
│ Notifications│            │ Inv.    │  │ Notif.  │
└──────────────┘            └─────────┘  └─────────┘
```

Benefits:
- Independent deploy cadence.
- Different DBs/runtimes per service.
- Failure isolation (payments outage doesn't kill catalog).

Costs:
- Network latency between services.
- Distributed transactions become Saga + Outbox.
- Operational complexity (deploys, monitoring, tracing).

Rule of thumb: split when teams are larger than ~8 engineers per service, or when domains have different scale/latency profiles.

### Z-Axis: Sharding

Same code, different data. Each instance owns a subset (a *shard*) of the data.

```
Shard 1: user_id % 4 == 0
Shard 2: user_id % 4 == 1
Shard 3: user_id % 4 == 2
Shard 4: user_id % 4 == 3
```

Needed when a single DB node can't hold the working set or serve QPS. Common shard keys: `user_id`, `tenant_id`, geographic region.

See [Sharding & Partitioning](../building-blocks/sharding-partitioning.md) for details.

### Vertical Scaling

"Bigger box." More CPU, more RAM, faster disk. Limits:

- Cost grows super-linearly past commodity hardware.
- Single failure domain.
- Eventually you hit physical ceiling (largest available SKU).

Right tool for: databases that can't shard, in-memory caches, single-leader systems.

### Stateless vs Stateful

| Stateless | Stateful |
|-----------|----------|
| Easy to scale horizontally | Sticky sessions / consistent hashing required |
| Any node can serve any request | Specific nodes own specific data |
| Examples: API servers, render workers | DB nodes, cache shards, Kafka brokers |

Rule: keep the *application* tier stateless; push state to a dedicated tier that's designed for it.

### Read Scaling vs Write Scaling

Different problems:

**Read scaling** (easier):
- Read replicas.
- Caching (Redis, CDN).
- Materialized views.
- Eventual consistency where acceptable.

**Write scaling** (harder):
- Sharding.
- Write-behind buffering.
- Batching.
- Split into multiple writable services (e.g., separate write paths for orders vs activity log).

A read-heavy app (10:1, 100:1) is much easier to scale than a write-heavy app.

### Amdahl's Law

```
speedup = 1 / ((1 - P) + P/N)
```

Where P is the parallelizable fraction. Sobering: even 95% parallel work has a hard ceiling at 20× speedup. The 5% serial bottleneck dominates eventually.

In practice: any **single point** (a primary DB, a global lock, a coordinator) eventually caps your scale, no matter how many app servers you add.

### Universal Scalability Law (Gunther)

Refines Amdahl with **coherence cost**: as N grows, coordination between nodes pulls throughput *down*. Plotted, throughput peaks and then declines past an optimum.

Means: more boxes is not always more throughput. Past a point, you're paying more for coordination than you gain in capacity.

### Designing for Scale Up-Front

A pragmatic recipe:

1. **Make the app stateless.** Any future scaling is impossible without this.
2. **Externalize sessions.** Redis or signed JWTs.
3. **Idempotent endpoints.** Retry-safe by default (`Idempotency-Key`).
4. **Choose a shardable data model.** Even if you start with one DB, design rows around a future shard key.
5. **Async where possible.** Queue heavy work; don't block request threads.
6. **Cache strategically.** Identify the 20% hot data; cache that.
7. **Measure before scaling.** Profile p99, find the bottleneck, scale *that* layer.

### Common Mistakes

| Mistake | Reality |
|---------|---------|
| Scaling app tier when DB is the bottleneck | Move the bottleneck; don't pile on the wrong layer |
| Sharding too early | Sharding adds complexity for life; defer until you have to |
| Microservices on day one for a 5-person team | Bigger ops burden than monolith; do Y-axis only when teams demand it |
| Caching everything for "performance" | Cache adds invalidation bugs; cache hot data only |
| Assuming horizontal scale = infinite | Universal Scalability Law says no |

> [!NOTE]
> Scale problems are rarely about adding boxes — they're about removing coordination. The fastest single-shard system you can design beats the largest distributed system you can build.

### Interview Follow-ups

- *"What's the biggest single bottleneck in your design?"* — Always have an answer; that's where you'd scale next.
- *"Can the DB handle 10× the QPS?"* — If no: read replicas + cache for reads; sharding for writes.
- *"How would you scale to global users?"* — Multi-region deploy, latency-routed DNS / Anycast, regional caches, eventual consistency for non-critical data.
