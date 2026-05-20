# Q: Redis vs Memcached — picking a distributed cache.

**Answer:**

The two dominate distributed caching. Memcached is older, simpler, faster on pure key-value. Redis is richer, with data structures, persistence, pub/sub, scripting. Modern default: **Redis** unless you need pure speed and nothing else.

### Memcached

Pure in-memory key-value store. Multi-threaded.

```
SET key value 0 60        # value, flags, expiry
GET key
DELETE key
```

- One data type: opaque blob.
- LRU eviction.
- No persistence.
- Multi-threaded (scales on a single box with many cores).
- Sharding: client-side consistent hashing.
- Operations strictly per-key; no multi-key transactions.

Strengths:
- Simplest semantics.
- Fastest pure get/set.
- Predictable memory model.

### Redis

In-memory data-structure store. Single-threaded core + I/O threading.

Data types:
- Strings (incl. counters).
- Lists.
- Sets, Sorted Sets.
- Hashes.
- Streams.
- Bitmaps, HyperLogLog, Geo.

Operations:
- Transactions (MULTI / EXEC).
- Lua scripting (atomic).
- Pub/Sub.
- Streams (consumer groups, like a mini Kafka).
- Persistence (RDB snapshots + AOF append-only file).
- Replication + Cluster mode.

Strengths:
- Much more than a cache (queues, leaderboards, locks, rate limiters).
- Persistence option.
- Richer client libraries.

### When To Pick Which

| Need | Pick |
|------|------|
| Cache only, KV, max throughput | Memcached |
| Counters, leaderboards, queues | Redis |
| Rate limiting, distributed locks | Redis |
| Pub/Sub fanout | Redis |
| Streaming with retention | Redis Streams (or Kafka) |
| Persistence required | Redis |
| Single-thread perf is enough, multi-core not critical | Either |

For 90% of new projects: Redis. Memcached's niche is rare these days.

### Persistence (Redis)

- **RDB**: snapshot to disk every N seconds. Fast restore; some loss.
- **AOF**: append every write to a log. Slower; minimal loss (configurable fsync).
- **Hybrid**: AOF for durability + RDB for fast restore.

For pure cache: persistence off; treat as ephemeral.
For session store / queue / counter: AOF with `everysec` fsync — good balance.

### Replication

```
master → slave (or in modern terms, primary → replica)
```

Async by default. Replica serves reads (optionally); failover if primary dies.

Redis Sentinel monitors and orchestrates failover.

### Redis Cluster

16,384 hash slots assigned across nodes. CRC16(key) → slot → node.

```
3-master, 3-replica cluster:
  M1 (slots 0-5460)        ← R1
  M2 (slots 5461-10922)    ← R2
  M3 (slots 10923-16383)   ← R3
```

Client gets `MOVED` redirect if it lands on wrong node. Multi-key operations require keys in same slot (use `{}` hash tags: `user:{42}:profile` and `user:{42}:settings` share a slot).

### Memcached Sharding

Pure client-side. The client library hashes the key and picks the node:

```
node = consistent_hash(key) → server in ring
```

No coordinated state. Adding a node moves ~1/N of keys.

### Throughput Benchmarks

Single node:
- Memcached: ~1.5M ops/sec/core (multi-threaded).
- Redis: ~100k–200k ops/sec single-threaded; more with pipelining + io_uring.

Multi-core scaling:
- Memcached scales linearly across cores in one process.
- Redis traditionally requires multiple instances (or use KeyDB, Dragonfly for multi-core Redis-protocol-compatible).

### Pipelining

Both support pipelining: send N requests in one network round trip.

```
pipeline.set(a, 1).set(b, 2).get(c).execute()
```

10× throughput improvement under high QPS. Critical for production tuning.

### Eviction

- Memcached: LRU per slab class.
- Redis: configurable (`maxmemory-policy`):
  - `noeviction`: reject writes when full.
  - `allkeys-lru`, `allkeys-lfu`, `allkeys-random`.
  - `volatile-lru`, etc. (only keys with TTL).

For pure cache: `allkeys-lru` typical. For mixed workload (cache + counters that must not be evicted): set TTL only on cacheable keys and use `volatile-lru`.

### Operational Concerns

- **Memory fragmentation**: monitor `used_memory_rss / used_memory`. Restart if high.
- **Persistence cost**: AOF fsync can stall the event loop.
- **Big keys**: a 1 GB string is fine to fetch once; killing if accidentally enumerated. Avoid.
- **SLOWLOG**: identify slow commands.
- **Cluster slot reshard during traffic**: smooth via streaming; brief blips.

### Common Mistakes

| Mistake | Fix |
|---------|-----|
| Using `KEYS *` in production | O(N) blocking; use `SCAN` |
| Storing huge JSON blobs as one string | Slow ops; split into Hash fields |
| No connection pooling | Per-request connect overhead |
| Single Redis primary as SPOF | Sentinel + replica; failover plan |
| Using Redis as primary DB without durability | Data loss on restart |

### Alternatives

- **KeyDB**: multi-threaded Redis fork.
- **Dragonfly**: modern Redis-compatible with much higher throughput.
- **Hazelcast / Ignite / Aerospike**: enterprise in-memory data grids.
- **Cloud-managed**: ElastiCache, MemoryDB (durable Redis), Memorystore.

For pure cache at scale: Memcached or Dragonfly. For everything else: Redis.

> [!NOTE]
> The choice often comes down to ecosystem comfort and team familiarity. Both work. Pick the one that matches your durability + data-structure needs.

### Interview Follow-ups

- *"How do you ensure consistency between Redis cache and DB?"* — Cache-aside pattern with delete-on-write + TTL. Never set the cache on write (race).
- *"How would you build a distributed lock in Redis?"* — `SETNX` with TTL; Redlock for multi-node Redis (controversial — see Martin Kleppmann's critique). For most cases, single-instance SETNX is fine.
- *"What about MemoryDB?"* — AWS's durable Redis (multi-AZ persistence). For workloads that need Redis API + DB-grade durability.
