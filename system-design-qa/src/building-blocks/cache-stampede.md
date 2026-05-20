# Q: Cache stampede / thundering herd — what causes it and how to fix it.

**Answer:**

A **cache stampede** happens when a hot key expires (or a cache restarts) and many concurrent requests stampede the origin, often crushing the database. Easy to overlook; can take down production.

### The Failure

```
t=0:    100k QPS hits cache; 99% hits.
t=10:   hot key expires.
t=10s:  100k requests all miss; all hit DB simultaneously.
t=10s:  DB collapses under the load it normally never sees.
```

The hotter the key, the worse the stampede.

### Causes

1. **Hot key expires**: TTL hits; everyone races to refill.
2. **Cache restart**: Redis crash + reboot loses all keys.
3. **Cache deploy / scaling event**: new nodes start cold.
4. **Mass invalidation**: a tag purge nukes thousands of keys at once.

### Mitigations

#### 1. Locking / Request Coalescing

Only one request fetches; others wait:

```python
def get(key):
    val = cache.get(key)
    if val is not None:
        return val

    if cache.setnx("lock:" + key, "1", ttl=10):       # try acquire
        try:
            val = db.get(key)
            cache.set(key, val, ttl=600)
            return val
        finally:
            cache.delete("lock:" + key)
    else:
        # Someone else is refreshing — wait briefly
        time.sleep(0.1)
        return get(key)        # retry
```

The first miss takes the lock; the rest poll. DB sees one request.

Variants:
- Java: Caffeine's `LoadingCache` does this in-process.
- Go: `singleflight`.
- Distributed: Redis SETNX or Redlock.

#### 2. Stale-While-Revalidate

Serve stale during async refresh:

```
val, age = cache.get_with_age(key)
if age > TTL:                       # stale
    async_refresh(key)               # background
return val                           # serve stale anyway
```

The user sees no latency spike. Origin sees one refresh, not many.

Browser / CDN support this via `Cache-Control: stale-while-revalidate=...`.

#### 3. Probabilistic Early Expiration

Each client treats TTL as random within a window:

```
effective_ttl = ttl × (1 + random()×0.1)
```

Or smarter: XFetch algorithm — refresh probability rises as TTL nears:

```
on read:
   if random() < (now - expiry) / refresh_window:
       refresh()
   return cached_value
```

Spreads refresh across clients; only one wins the race naturally.

#### 4. Permanent Cache + Background Refresh

Don't expire; refresh periodically. Cache is always populated; just possibly stale.

```
on read:        return cache.get(key)
every 60s:      cache.set(key, db.get(key))
```

Works for read-mostly data with bounded staleness budget. No stampede ever, because no expiry.

#### 5. Pre-Warming on Deploy

Before promoting a new cache node, pre-populate it from the old:

```
new node joins cluster
   → drain replica or fetch from sibling
   → mark ready when caught up
```

Avoids cold-start stampede.

#### 6. Hot Key Replication

For a single key with massive traffic:
- Replicate the key to all cache nodes (read-anywhere).
- Or process-local L1 cache for that key only.
- Reduces request to a single shard.

### Stampede Scenarios by Scale

| Scenario | Risk |
|----------|------|
| Many low-traffic keys, all TTL aligned | Spike at the boundary — jitter the TTLs |
| One celebrity-user key | Hot stampede — lock + warm |
| Cache restart at 9 AM (deploy time) | Cold stampede — gradual warmup |
| Tag-based purge of 10k keys | Bulk stampede — rate-limit purge |

### Detection

Metrics:
- Backend QPS spike correlated with cache miss spike.
- p99 latency briefly degraded.
- Lock contention metrics (if you're using locks).

Alert on:
- `cache_misses_per_sec` spike > N× baseline.
- DB request count anomaly.

### Common Mistakes

| Mistake | Fix |
|---------|-----|
| All keys with same TTL | Boundary stampede; jitter |
| No lock on cache miss | Every concurrent request hits DB |
| Trying to lock with `GET+SET` (race) | Use atomic `SETNX` |
| Locking forever (lock never released on crash) | Lock TTL + heartbeat |

> [!NOTE]
> Cache stampedes are silent killers. They don't show up under steady state — only when the cache is "doing its job" most of the time and a hot key happens to expire. Test stampede behavior explicitly (drop a hot key, watch the origin).

### Interview Follow-ups

- *"What's the cheapest fix?"* — TTL jitter + SWR. No new infra, minimal code.
- *"How does Caffeine handle this in-process?"* — `LoadingCache` with a `CacheLoader`; only one thread computes per key; others wait on the result.
- *"How would you handle a cold-start stampede after Redis restart?"* — Replay-from-replica + canary connection draining + connection-pool limit toward DB (force backpressure).
