# Q: Design a distributed cache (memcached / Redis Cluster).

**Answer:**

The interview tests whether you understand consistent hashing, replication, eviction, and failure modes — not whether you can write a from-scratch cache server.

### Requirements

**Functional**:
- Key-value `GET` / `SET` / `DELETE`.
- TTL support.
- Optional: increment counters, bulk ops.

**Non-functional**:
- p99 < 5 ms.
- 1M+ QPS aggregate.
- Memory-bounded; evict on pressure.
- Survive node loss without data loss for cache (cache may rebuild from origin).

### Choices Up Front

**Single-node vs distributed**: at > ~100 GB working set or > 100k QPS, you need a cluster.

**Persistence**: pure cache → no. As primary store → yes (Redis AOF / RDB).

**Replication**: critical for availability, optional for cache (warm-up acceptable).

### High-Level Architecture

```
                  Clients
                     │
                     ▼
                ┌─────────┐
                │  Proxy  │   optional layer (twemproxy / envoy)
                └────┬────┘
                     │
       ┌─────────────┼─────────────┐
       ▼             ▼             ▼
   ┌─────┐       ┌─────┐       ┌─────┐
   │ N1  │       │ N2  │       │ N3  │     ← cache nodes
   └─────┘       └─────┘       └─────┘
   primary       primary       primary
       │             │             │
       ▼             ▼             ▼
   ┌─────┐       ┌─────┐       ┌─────┐
   │ R1  │       │ R2  │       │ R3  │     ← replicas
   └─────┘       └─────┘       └─────┘
```

Each node owns a portion of the key space (consistent hashing). Replicas serve as failovers (and optionally reads).

### Key Distribution

**Consistent hashing** (see [Consistent Hashing](../patterns/consistent-hashing.md)). Each node owns multiple vnodes on the ring. Adding/removing a node moves only `1/N` of keys.

Redis Cluster: 16,384 hash slots distributed across nodes. CRC16 of key → slot → node. Re-sharding moves slots.

Client-side hashing: cache client library knows the topology; routes directly. No proxy needed but the client is fatter.

Proxy-side hashing: clients connect to a thin proxy that routes. Easier to manage topology; one more hop.

### Node Internals

Each node:
- Hash map (key → value).
- LRU/LFU list for eviction.
- Expiration: lazy (check on read) + active (periodic scan).
- Networking: epoll/io_uring, single-threaded event loop (Redis) or per-core sharding (KeyDB, Dragonfly).

Memory layout matters at scale:
- Use slab allocators (memcached) to avoid fragmentation.
- Pack small values inline; large values pointer-referenced.

### Eviction Policies

When memory full:
- **LRU**: evict least recently used. Default for most caches.
- **LFU**: evict least frequently used. Better when access patterns are skewed.
- **Random**: surprisingly effective; cheap.
- **TTL-aware**: evict the soonest-to-expire first.
- **No-eviction**: reject writes (used for queues / counters).

Redis supports approximate LRU/LFU (samples a few keys, evicts the least-good among them).

### Replication

Async master → replica replication.

- Write goes to primary.
- Primary streams changes to replica.
- On primary loss, replica promoted.

For cache: replicas mainly for failover, sometimes for read scaling. Stale reads possible (replication lag); usually fine for cache.

### Failure Detection & Failover

- **Gossip**: every node exchanges health info with peers (Redis Cluster does this).
- **Sentinel / external coordinator**: separate process monitors and orchestrates failover.

On primary failure:
- Quorum agrees primary is down.
- Promote replica.
- Tell clients to re-route (slot map update).

### Consistency

Cache is **eventually consistent** by default:

- Lossy on node failure (data not yet replicated).
- Race conditions on cache-aside (delete vs concurrent fill).

For strict consistency, you'd need:
- Synchronous replication (slow).
- Distributed locks for write paths (slower).

Most caches accept eventual consistency for performance.

### Cache Stampede / Thundering Herd

When a hot key expires, hundreds of clients hit the origin simultaneously.

Mitigations:
- **Lock + fetch**: first miss takes the lock, others wait.
- **Refresh-ahead**: refresh near expiry, in the background.
- **Stale-while-revalidate**: serve stale during async refresh.
- **Probabilistic early expiration**: each client treats TTL with random jitter; one of them refreshes early.

See [Cache Stampede](../building-blocks/cache-stampede.md).

### Sharding Topology

For 1 TB cache:

```
cluster: 12 nodes × ~100 GB each
replicas: each node has 1 replica (24 instances total)
slots: 16,384 distributed evenly
network: 10 Gbps per node
```

Add capacity: bring up new node → re-slot → cluster rebalances live.

### Memory Sizing

Pick by working set, not by total data:

```
hot set = top 20% of keys
if hot set fits in RAM → hit rate ≈ 99%
```

Track: `evictions/sec`, `hit_ratio`. Rising evictions = working set outgrowing memory.

### Client Library

Cache client must:
- Cache slot → node map.
- Re-route on `MOVED` (Redis) / `SERVER_ERROR` responses.
- Pool connections per node.
- Pipeline batched requests.
- Circuit-break on dead nodes.

### Hot Key Mitigation

One key takes 80% of QPS (celebrity user's session, viral post).

Options:
- **Replicate the key across all nodes** (Redis Cluster doesn't support natively; do at app layer).
- **Client-side fanout**: read from any of N replicas of the hot key.
- **Local cache layer in front** (process-local L1).

### Common Mistakes

| Mistake | Fix |
|---------|-----|
| Using cache as primary store without persistence | Data loss on restart |
| Same DB connection pool size as cache QPS | Cache miss flood crushes DB |
| Big values (> 100 KB) in Redis | Latency spikes; split or move to S3 |
| No metrics on hit ratio | Don't know if cache is doing work |
| Cache invalidation across many nodes | Use TTLs + delete-on-write |

> [!NOTE]
> The core of a distributed cache is two ideas: **consistent hashing for routing** and **eventual consistency for replication**. Most other features (TTL, eviction, pipelining) are local-node concerns.

### Interview Follow-ups

- *"How does Redis Cluster handle a node-add?"* — Move slots from existing nodes to the new one in batches; clients get `MOVED` redirects during the migration.
- *"How would you build a cache for sub-microsecond reads?"* — Off the network; use in-process LRU (Caffeine for JVM, lru-cache for Node) backed by a distributed cache for warmup.
- *"How do you cache binary blobs?"* — Either small (< 100 KB) in Redis, or pointer + S3 URL stored in cache, blob in S3 with CDN in front.
