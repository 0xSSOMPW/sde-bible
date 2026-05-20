# Q: Explain consistency models — strong, eventual, causal, read-your-writes, monotonic.

**Answer:**

"Consistency" is overloaded — it means one thing in CAP, another in ACID, and a third in distributed-systems literature. The literature definition is the one that matters at design time: **what guarantees does a read make about the writes it observes?**

### The Spectrum

```
Strongest                                                           Weakest
─────────────────────────────────────────────────────────────────────────►

Strict Serializable   Linearizable   Sequential   Causal   Eventual
```

Roughly: the further left, the more coordination required and the slower the system.

### Strong Consistency (Linearizability)

A read returns the **most recent committed write** in real time. As if there were one copy of the data and operations happened one at a time.

```
T1: write(x, 1)
T2: write(x, 2)         (after T1 commits)
T3: read(x)             must return 2
```

Achieved via consensus (Raft/Paxos), single-leader writes with synchronous replication, or external clocks (Spanner).

Cost: latency = round-trip to quorum. Cannot be globally faster than the speed of light × distance.

### Sequential Consistency

Operations appear in **some total order** consistent with each process's program order, but not necessarily the real-time order.

Weaker than linearizable. Strong enough for many algorithms (caches, mutexes).

### Causal Consistency

Operations that are **causally related** are seen in the same order by all replicas. Concurrent operations can be observed in any order.

```
T1: post(p1)
T2: comment(p1, c1)         depends on p1
                            → every reader sees p1 before c1
T3: post(p2)                concurrent with above
                            → readers may see p2 before or after p1/c1
```

Implemented with vector clocks. Practical for social feeds, collaborative editing.

### Read-Your-Writes

A user always sees their own writes. Other users may not see them yet.

```
You post a photo.
You immediately refresh — you see the photo.
Your follower refreshes — may not see it yet.
```

Common pattern: route a user's reads to the same replica (or leader) as their writes for a short window. Or buffer the write client-side until visible.

### Monotonic Reads

If a process reads value `v1`, subsequent reads will see `v1` or newer. Never goes backward.

Without monotonic reads, a load-balanced read might bounce between replicas with different lag and the user sees state regress. Stick a user to a replica (sticky sessions) to fix.

### Monotonic Writes

A process's writes are applied in the order it issued them. Often paired with read-your-writes.

### Eventual Consistency

Given no new writes, all replicas converge to the same value. No bound on *when*.

Strongest claim: "eventually" — useless for SLAs unless you add bounded staleness.

```
Cassandra Read One:    nearest replica wins, may be stale by seconds
DynamoDB eventual:     up to ~1s lag typical
S3:                    strong consistency since 2020
```

### Bounded Staleness

A practical strengthening of eventual: read no more than `T` seconds (or `N` operations) behind the latest committed write.

Cosmos DB exposes this as a knob: "5 seconds" or "100 writes" max lag. Useful for dashboards that can be slightly behind but not arbitrarily so.

### Choosing Per Data Class

| Data | Right model |
|------|------------|
| Bank balance | Linearizable |
| Order status | Read-your-writes minimum |
| Inventory decrement | Linearizable on the SKU |
| Chat message order in a room | Causal |
| User profile read after edit | Read-your-writes |
| Social like count | Eventual |
| Search result | Eventual w/ bounded lag |
| Analytics dashboard | Bounded staleness |

### Where Each Lives

| System | Default model |
|--------|--------------|
| Postgres (single leader) | Linearizable on the leader; standby is async eventual |
| MySQL semi-sync | Linearizable on commit; standby slightly behind |
| Spanner | Strict serializable (linearizable + serializable txns) |
| DynamoDB | Eventual default, strong on opt-in |
| Cassandra | Tunable per query (`ONE`, `QUORUM`, `ALL`) |
| Redis | Linearizable on the primary; replicas async |
| Kafka per partition | Linearizable per partition |

### Read-Your-Writes Implementations

**1. Sticky session.** Send each user to the leader (or same replica) for a window after a write.

**2. Version pinning.** Client stores the write's version; reads include `min_version`, server waits or routes to a replica that has it.

**3. Client-side cache.** App holds the just-written value locally until the server's view catches up.

### Causal Consistency Implementations

**1. Vector clocks.** Each replica tags writes with a vector of per-node counters. Reader walks dependencies.

**2. Lamport clocks + dependency tracking.** Simpler but less precise.

**3. CRDT (conflict-free replicated data types).** Some CRDTs are causal by construction.

### Common Mistakes

| Mistake | Fix |
|---------|-----|
| Treating "eventually consistent" as "eventually we'll fix the bug" | Set a measurable bound; treat it as an SLI |
| Read-your-writes via "wait 500 ms" | Brittle; use sticky routing or version pinning |
| Causal consistency for monetary writes | Underpowered; use linearizable |
| Strong consistency for everything | Latency floor = cross-region RTT |

> [!NOTE]
> In interviews, when asked "what consistency does this need?", the wrong answer is "strong" by default. The right answer is "linearizable for X, causal for Y, eventual for Z, with bounded staleness of ~T."

### Interview Follow-ups

- *"What's the difference between linearizable and serializable?"* — Linearizable: real-time order on a single object. Serializable: transactions equivalent to some serial schedule. Strict serializable = both.
- *"How does Spanner achieve external consistency?"* — TrueTime (GPS + atomic clocks bound clock skew); commits wait out the uncertainty window.
- *"Why is eventual consistency cheap?"* — No coordination on the write path; replicas reconcile in the background.
