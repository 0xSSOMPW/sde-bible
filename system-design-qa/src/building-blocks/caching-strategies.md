# Q: Cache strategies — cache-aside, write-through, write-back, write-around, read-through.

**Answer:**

A cache speeds up reads (sometimes writes) by sitting in front of a slower store. The five canonical strategies differ in **who** writes to the cache vs DB, and **when**. Pick by failure mode you can tolerate.

### The Five Strategies

| Strategy | Read path | Write path | Failure if cache dies |
|----------|----------|-----------|----------------------|
| Cache-aside (lazy) | App: check cache → miss → fetch DB → fill cache | App writes DB; invalidates cache | DB takes full load |
| Read-through | App reads cache; cache fetches from DB on miss | App writes DB directly | App still reads DB if cache off |
| Write-through | Same as read-through | App writes cache; cache writes DB sync | Slower writes if cache off |
| Write-back (write-behind) | Same as read-through | App writes cache; cache flushes DB async | Data loss risk if cache dies before flush |
| Write-around | Reads as above | App writes DB only; cache filled lazily | Cold reads of new data |

### Cache-Aside (Most Common)

```python
def get(key):
    val = cache.get(key)
    if val is None:
        val = db.get(key)
        cache.set(key, val, ttl=600)
    return val

def update(key, val):
    db.put(key, val)
    cache.delete(key)         # invalidate; don't update
```

Properties:
- Simple, app-controlled.
- Cache failures degrade gracefully (DB serves).
- **Stale window** between DB write and cache invalidate.
- Cache misses go to DB — stampede risk under burst load.

Why `delete` instead of `set` on writes: if two writes race, deletes are commutative; `set` can install a stale value.

### Read-Through

Cache exposes the same API as the DB; the cache itself does the DB lookup on miss.

```
get(key):
  cache layer:
    val = local.get(key)
    if val is None:
        val = db.get(key)
        local.set(key, val, ttl)
    return val
```

Centralizes cache logic, simplifies the app. Most managed caches (AWS DAX) work this way.

### Write-Through

Writes go to cache, cache writes DB synchronously.

```
write(key, val):
  cache.set(key, val)
  db.put(key, val)        # sync
  return success
```

Pros: cache always in sync.
Cons: writes are slower (cache + DB latency). Cache must be in front of every write path — no out-of-band DB writes allowed (or you'll have drift).

### Write-Back / Write-Behind

Writes to cache only; cache flushes to DB asynchronously (batched).

```
write(key, val):
  cache.set(key, val)
  schedule async DB write
  return success            ← fast
```

Pros: ultra-fast writes, batching for DB efficiency.
Cons: **data loss** if cache dies before flush. Suitable for metrics, counters, analytics. Not for financial data.

### Write-Around

Writes bypass cache, only DB is touched. Cache is filled on subsequent reads.

```
write(key, val):
  db.put(key, val)
  # do NOT write cache

read(key):
  val = cache.get(key)
  if val is None:
      val = db.get(key)
      cache.set(key, val)
```

Useful for write-heavy data that's rarely re-read (logs, telemetry).

### Choosing

| Workload | Pick |
|---------|------|
| Read-heavy, eventual freshness OK | Cache-aside |
| Read-heavy, want strict freshness | Write-through + invalidation |
| Write-heavy, durability critical | DB-only or cache-aside |
| Write-heavy, durability OK to lose | Write-back |
| Writes never re-read soon | Write-around |
| App should be ignorant of cache | Read-through |

### TTL — the Quiet Hero

Every cache entry should have a TTL. Reasons:
- Limits stale-data damage from bugs in invalidation.
- Bounds cache size organically.
- Keeps cache "honest" — if a key is wrong, it self-heals in TTL.

Typical TTLs:
- User profile: 5–60 min.
- Static reference data: hours/days.
- Anything counters or timeseries: seconds.

### Negative Caching

Cache "no, this doesn't exist" responses too. Otherwise a flood of requests for nonexistent keys (e.g., bots probing) hits the DB.

```
val = db.get(key)
if val is None:
    cache.set(key, NULL_SENTINEL, ttl=60)    # short TTL
```

### Refresh-Ahead

Refresh popular keys *before* they expire to avoid the miss stampede:

```
on read:
   val, ttl_remaining = cache.get_with_ttl(key)
   if ttl_remaining < 10% of original:
       async_refresh(key)
   return val
```

The user serving the request never waits on the refresh.

### Multi-Tier Caches

```
Browser cache  (HTTP cache headers)
   ▼
CDN            (edge, geo)
   ▼
App-server local cache  (in-process LRU)
   ▼
Distributed cache  (Redis cluster)
   ▼
DB
```

Each layer absorbs progressively more traffic. p99 latency is set by the cheapest cache that's still warm enough.

### Cache Hit Ratio

Most important cache metric.

```
hit_rate = hits / (hits + misses)
```

Healthy: > 80%. Excellent: > 95%. Below 50% — the cache is barely earning its keep; revisit keys, TTLs, capacity.

### Common Mistakes

| Mistake | Fix |
|---------|-----|
| Updating cache on write instead of deleting | Race conditions → stale values |
| No TTL "to maximize hit ratio" | Bugs become permanent; size grows unbounded |
| Caching everything | Adds invalidation surface; cache 20% hot keys |
| Caching huge objects | Network + CPU; consider field-level caching |
| One Redis for hot + cold data | Hot data evicts cold causing fetches |
| Identical TTL on all keys | Synchronous expiry → stampede; jitter TTLs |

> [!NOTE]
> The first rule of caching: invalidation is the hardest problem. The second rule: TTL is your friend — design assuming invalidation will sometimes fail.

### Interview Follow-ups

- *"How would you achieve strong consistency between cache and DB?"* — Hard. Options: write-through with cache as source of truth; or accept eventual + short TTLs; or use CDC to invalidate cache from DB changes.
- *"What about cache stampede on cold start?"* — See [Cache Stampede](./cache-stampede.md): request coalescing, locks, probabilistic early refresh.
- *"How do you decide what to cache?"* — Hot-set analysis: log key accesses, find the Pareto 20%. Cache that; ignore the rest.
