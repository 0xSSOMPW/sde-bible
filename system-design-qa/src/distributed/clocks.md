# Q: Distributed clocks — Lamport, vector, hybrid logical.

**Answer:**

Wall-clock time on different machines disagrees. Distributed systems use **logical clocks** to reason about event ordering without relying on synchronization. Three flavors: Lamport, vector, hybrid logical.

### Why Wall Clocks Fail

```
Node A's clock: 12:00:00.500
Node B's clock: 12:00:00.450

A: sends message at A's time 12:00:00.500
B: receives at B's time 12:00:00.450      ← appears to receive before A sent it
```

NTP keeps drift to ~10ms but can't eliminate it. Clocks even go *backwards* during sync adjustments.

### Lamport Clocks

Each process maintains a counter. Bump on event:

```
on local event:    counter += 1
on send msg:       counter += 1; attach counter to msg
on receive msg:    counter = max(counter, msg.counter) + 1
```

Properties:
- If event `a` happened-before `b`, then `clock(a) < clock(b)`.
- Reverse not necessarily true — `clock(a) < clock(b)` doesn't mean `a → b`.
- Total order: ties broken by node ID.

```
A: counter 0 → 1 (local)
A: counter → 2 (send to B with timestamp 2)
B: receives, sets counter = max(0, 2) + 1 = 3
```

Useful for total-ordering events into a single log. Used in Cassandra (last-write-wins timestamps), older distributed locks.

### Vector Clocks

Each process holds a vector of counters — one per process:

```
A: [Va, 0, 0]
B: [0, Vb, 0]
C: [0, 0, Vc]

on local event:    self[i] += 1
on send msg:       self[i] += 1; attach copy
on receive:        for each j: self[j] = max(self[j], msg[j]); self[i] += 1
```

Properties:
- `a → b` iff `clock(a) ≤ clock(b)` element-wise.
- `a || b` (concurrent) iff neither is ≤ the other.

Detects concurrent edits — critical for CRDTs, Dynamo-style conflict resolution.

Cost: O(N) size where N = number of nodes. Grows with cluster size.

### Hybrid Logical Clocks (HLC)

Combines wall-clock and logical counters:

```
HLC = (physical_time, logical_counter)

on local event:
  if physical_time > hlc.physical:
    hlc = (physical_time, 0)
  else:
    hlc.logical += 1
  
on send: attach hlc
on receive (msg):
  hlc.physical = max(local_physical, msg.physical, hlc.physical)
  hlc.logical  = max(... computed appropriately) + 1
```

Properties:
- Close to wall-clock time (humans recognize timestamps).
- Captures happened-before via the logical part.
- Compact (16 bytes typical).

Used in CockroachDB, YugabyteDB, MongoDB causal consistency.

### Comparison

| Clock | Order | Size | Use |
|-------|-------|------|-----|
| Lamport | Total (with tiebreak) | 8 bytes | Single-log ordering, LWW |
| Vector | Partial (concurrent detectable) | O(N) | CRDT, Dynamo conflict resolution |
| HLC | Near wall-clock + happened-before | 16 bytes | Distributed SQL (Cockroach), causal stores |

### Last-Write-Wins (LWW)

Cassandra and others: every write tagged with timestamp; on conflict, larger timestamp wins.

Problem: relies on wall-clock. Clock skew → "wrong" write wins. Lost data.

Mitigations:
- NTP discipline (still not perfect).
- HLC instead.
- Application-level vector clocks for conflict detection (then user resolution).

### TrueTime (Google Spanner)

Spanner uses GPS + atomic clocks across DCs to bound clock uncertainty. `TT.now()` returns `[earliest, latest]` interval.

Commit waits out the interval before exposing the write — guarantees external consistency without coordination.

Hardware + careful engineering. Most don't replicate it.

### Causal Consistency

A consistency model implementable with vector clocks or HLC:

```
op A → op B (causal)
all replicas see A before B
concurrent ops may be seen in either order
```

Strong enough for: collaborative editing, social feed ordering, replicated KV stores.

### Idempotency vs Time

For idempotency by event ID, you don't need synchronized clocks — just unique IDs (Snowflake, UUID).

For *ordering of events from the same producer*, clocks suffice (just local). For *cross-producer ordering*, you need logical clocks.

### Common Mistakes

| Mistake | Result |
|---------|--------|
| Using wall-clock for cross-process ordering | Skew bugs, dropped writes |
| Vector clocks at scale (thousands of nodes) | Vectors too big; switch to HLC |
| LWW on financial data | Silent data loss when clocks disagree |
| Trusting NTP for sub-second correctness | NTP isn't a guarantee; use logical clocks |

> [!NOTE]
> Wall clocks are advice; logical clocks are truth in distributed systems. If your design relies on "two machines agree on the current time," reconsider.

### Interview Follow-ups

- *"What's the difference between linearizability and causal consistency?"* — Linearizable enforces real-time order on a single object globally. Causal enforces happened-before only; concurrent ops have flexible order.
- *"How do you detect concurrent writes in Dynamo?"* — Each write carries a vector clock; reads return all versions whose vectors are incomparable; application or LWW resolves.
- *"How does Spanner avoid running 2PC for every read?"* — TrueTime-bound timestamps + serializable snapshot isolation; reads from any replica at a given timestamp are linearizable.
