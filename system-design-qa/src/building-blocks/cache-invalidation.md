# Q: Cache invalidation & eviction — the hard part of caching.

**Answer:**

"There are only two hard things in computer science: cache invalidation and naming things." — Phil Karlton. The invalidation problem is real because **cache and source-of-truth can diverge**, and detecting / repairing the drift is the bulk of the engineering.

### Invalidation Approaches

**1. TTL (time-based)**

Every key expires after N seconds. Simplest, most reliable. The cache self-heals; stale data has a bounded lifetime.

```
cache.set(key, value, ttl=600)
```

Trade: stale data lives up to TTL.

Pick TTL by:
- How sensitive the data is to staleness.
- How much load the origin can take if all cache entries expire at once.

**2. Write-through invalidation**

On every write to source, also update / delete cache:

```
write(key, value):
  db.put(key, value)
  cache.delete(key)         ← delete, don't set
```

Why delete vs set: race conditions. Two concurrent writers can set stale value last. Deleting is commutative.

Trade: every write path must remember to invalidate. Easy to miss in heterogeneous code.

**3. Event-driven (CDC) invalidation**

The DB streams its changes; a separate process invalidates cache entries based on the stream:

```
Postgres WAL → Debezium → Kafka → cache invalidator
```

Decouples invalidation logic from write paths. Works even if a write bypassed your app.

**4. Versioning / tagging**

Each cached entry includes a version or set of tags. Invalidation = bump version / purge tag.

```
cache.set("user:42:v3", value)            # version in key
cache.set_with_tags(key, value, tags=["user:42"])

# Invalidate:
cache.invalidate_tag("user:42")
```

Fastly / Cloudflare support tag-based purge for CDN.

### Eviction Policies (When Cache Full)

| Policy | Description | Good for |
|--------|------------|---------|
| **LRU** | Evict least-recently-used | General workload |
| **LFU** | Evict least-frequently-used | Skewed access (Pareto) |
| **TTL-aware** | Prefer expiring soonest | TTL-heavy workloads |
| **Random** | Evict random | Surprisingly effective; cheap |
| **TinyLFU** | Hybrid; Caffeine default | Mixed workloads |
| **No-eviction** | Reject writes when full | Counters, queues |

Redis defaults: `noeviction`. Change via `maxmemory-policy`.

### LRU Implementation

Doubly linked list + hash map:
- On access: move to head.
- On insert: add at head; if over capacity, remove tail.
- O(1) per operation.

LRU is the universal default. LFU often does better when access patterns have a long tail of "loved" keys, but is harder to implement (LFU classic doesn't decay; "windowed TinyLFU" fixes it).

### Hot Key Eviction Problem

A single hot key + many less-popular keys + LRU = the hot key keeps the popular content in cache → less-popular evicted constantly → unstable behavior.

Fix: LFU or LRU + size-tier (keep large objects separate).

### Negative Caching

Cache "no" answers too:

```
val = db.get(key)
if val is None:
    cache.set(key, NULL_SENTINEL, ttl=60)
```

Without it, repeated lookups for non-existent keys hit the DB. Bots and crawlers exploit this.

Short TTL — newly inserted keys should appear after a brief window.

### Refresh-Ahead

Refresh popular keys *before* expiry:

```
on read:
    val, ttl_remaining = cache.get_with_ttl(key)
    if ttl_remaining < 0.1 * original_ttl:
        async_refresh(key)        # background
    return val
```

User never waits on refresh. Reduces stampede risk.

### Stampede Protection

Hot key expires; many clients race to refill:

- Lock + fetch (first one wins; others wait).
- Probabilistic early expiration (each client treats TTL with jitter).
- Stale-while-revalidate (serve stale during async refresh).

See [Cache Stampede](./cache-stampede.md).

### Multi-Tier Invalidation

```
Browser cache (Cache-Control)
  ▼
CDN edge cache
  ▼
App-server local cache (Caffeine, lru-cache)
  ▼
Distributed cache (Redis)
  ▼
DB
```

Invalidating across all tiers is hard. Strategies:
- Tag-based purge at CDN (Fastly, Cloudflare).
- Short TTL at lower tiers.
- Bust local caches via pub/sub on invalidation event.

### CDN-Specific Invalidation

```
purge by URL:    invalidate single URL
purge by tag:    invalidate group
soft purge:      serve stale until origin returns fresh
```

Purges are not instant — usually < 30s propagation.

### Avoid: Cache Aside With Concurrent Updates

```
client A: write DB
client B: write DB
client A: invalidate cache
client B: invalidate cache
client A: SET cache = old A read  ← race
```

This is why we *delete* rather than *set* on writes. The next miss reads from DB and fills correctly.

### Common Mistakes

| Mistake | Fix |
|---------|-----|
| `SET` cache on write | Race conditions; use `DELETE` |
| TTL = forever "for hit ratio" | Stale forever when bugs strike |
| Invalidating only on success | DB write succeeds, cache stays stale on app crash |
| No metrics on stale serving | Can't detect drift |
| Cache hit ratio target without observability | Optimizing the wrong number |

### Observability

- **Hit ratio**: `hits / (hits + misses)`.
- **Stale rate**: how often was the cached value out of date when re-fetched?
- **Eviction rate**: indicates capacity vs working set.
- **Invalidation events / sec**: too many = bug in upstream.

> [!NOTE]
> Treat invalidation as a separate, observable concern from the cache itself. The fact that "we have a cache" doesn't tell you whether it's correct — instrument what matters.

### Interview Follow-ups

- *"How do you keep cache consistent with DB?"* — You don't keep them perfectly consistent. Pick: TTL (bounded staleness), write-through (synchronous cost), or CDC (eventual). All have trade-offs.
- *"What's the right TTL?"* — Long enough that the cache hits; short enough that stale damage is bounded. Measure both.
- *"How do you cache personalized data?"* — Per-user key prefix; smaller hit ratios but precise. Or cache fragments and assemble at request time.
