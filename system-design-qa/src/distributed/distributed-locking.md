# Q: Distributed locks — how to build them, when to use them, when not to.

**Answer:**

A distributed lock makes one node "the only one doing X" across a cluster. It's tempting and dangerous in equal measure. Get it wrong and you have data corruption; get it right and you've added latency and a coordinator dependency.

### When You Want a Distributed Lock

- Singleton workers (only one runs the migration).
- Per-resource serialization (only one process writes to file X).
- Coordinated leader work.

### When You Don't

- Hot path (every request acquires lock → serialization bottleneck).
- Replacing transactions in your DB (use the DB's locking).
- Idempotency (use idempotency keys instead — no coordination needed).
- "Just in case" — every lock is a SPOF you didn't have before.

### Implementation Options

**1. Redis SETNX (single instance)**

```
SET lock:key unique_value NX EX 30
   - NX: only if not exists
   - EX 30: TTL 30 seconds
```

Release:

```lua
-- Atomic Lua: delete only if I own it
if redis.call('GET', KEYS[1]) == ARGV[1] then
    return redis.call('DEL', KEYS[1])
end
return 0
```

The `unique_value` (request UUID) prevents Bob from deleting Alice's lock.

Properties:
- Single point of failure (Redis goes down → no lock).
- Lease-based via TTL — handles holder crash.
- Cheap, ~ms latency.

Best for: short-lived locks (< 30 s) where SPOF risk is acceptable.

**2. Redlock (multi-instance Redis)**

Acquire lock from majority of N independent Redis instances:

```
for each redis in [r1, r2, r3, r4, r5]:
    try acquire with TTL T
   
if majority acquired AND total time < T:
    lock held
else:
    release everywhere and retry
```

Goal: tolerate Redis instance failures.

Famous critique: Martin Kleppmann (2016) argues Redlock lacks correctness guarantees under clock skew / GC pauses. Antirez (Redis author) disagrees. For most use cases, **Redlock is fine if you have fencing tokens downstream**.

**3. etcd / ZooKeeper / Consul**

Backed by Raft. Linearizable lease management.

etcd:

```python
lease = etcd.lease(ttl=30)
ok = etcd.put("lock:key", "owner", lease=lease.id, prevExist=False)
if ok:
    # I hold the lock
    do_work()
    lease.revoke()
```

Properties:
- Linearizable.
- Survives failures up to quorum.
- Higher latency than Redis (~10s of ms).
- Operates as both lease store + watcher.

Best for: correctness-critical locks where a few ms latency is acceptable.

**4. Database lock**

```sql
-- Postgres advisory lock
SELECT pg_try_advisory_lock(hash_key);
   -- ... do work ...
SELECT pg_advisory_unlock(hash_key);

-- Or for a row:
BEGIN;
SELECT * FROM resources WHERE id = ? FOR UPDATE;
   -- ... do work ...
COMMIT;
```

Properties:
- DB is already in the trust set.
- Limited by DB capacity.
- No TTL on advisory locks → holder crash holds lock until session ends.

### The Lock Holder Dies Problem

Critical: what if the lock holder crashes mid-work?

- Without TTL: lock held forever.
- With TTL: lock expires, someone else takes over. But original holder might still be doing work (slow GC, paused process, network blip). Two holders. Bug.

Solution: **fencing tokens.**

### Fencing Tokens

Lease grant includes a monotonically increasing token. Every downstream operation includes the token. Downstream rejects writes with stale tokens.

```
Alice acquires lock → fence=10
Alice's GC pauses 30s
Alice's lock expires
Bob acquires lock → fence=11
Bob writes to DB with fence=11

Alice resumes, tries to write with fence=10
DB sees fence < current → reject
```

Without fencing tokens, distributed locks can't be correct. Period.

Implementing: etcd revision, ZK zxid, Kafka epoch — all support fence-like semantics.

### Locking Latency

- Redis single-instance: 1–2 ms.
- Redlock: 5–20 ms (depending on cluster geometry).
- etcd: 10–50 ms.
- DB advisory lock: depends on DB load.

Hot path that needs 1k+ QPS with a lock → wrong design. Restructure to avoid coordination.

### Lock Granularity

- **Coarse**: one lock for all updates to a resource. Simple; reduces concurrency.
- **Fine**: one lock per row / key. More concurrent; more lock churn.

Match granularity to contention. Lock the smallest scope necessary.

### Avoiding Locks Entirely

Better alternatives when applicable:

**1. Optimistic concurrency** (version columns).

```sql
UPDATE row SET ..., version = version + 1
WHERE id = ? AND version = ?
```

No lock; retry on mismatch.

**2. Single-writer per partition** (Kafka).

If one entity has one partition, only one consumer processes it. No lock needed.

**3. Idempotency**.

If retries are safe, you don't need to prevent them.

**4. CRDTs**.

Mergeable types eliminate conflict resolution.

### Common Mistakes

| Mistake | Result |
|---------|--------|
| No TTL on lock | Holder crashes; lock held forever |
| Releasing lock without ownership check | One process deletes another's lock |
| Single Redis instance lock for critical data | SPOF; use Redlock or etcd |
| No fencing token downstream | Two holders silently corrupt |
| Lock + I/O | Holds lock during slow network call; massive contention |
| Locking around long batch job | Use a state machine, not a lock |

> [!NOTE]
> Distributed locks are a sharp tool. Two-line code with subtle correctness implications. Use them sparingly; prefer designs that don't need them.

### Interview Follow-ups

- *"How do you implement a lock with retry?"* — `acquire()` retries on contention with backoff + jitter; give up after max attempts; alert if pattern persists.
- *"What if a follower in etcd thinks it's leader?"* — Raft ensures only the elected leader can perform consistent reads/writes; lease grants are linearizable.
- *"Redlock or etcd for production?"* — etcd if you need provable correctness; Redlock if you have fencing tokens and Redis is already in the stack.
