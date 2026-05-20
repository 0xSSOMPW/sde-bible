# Q: Two-phase commit (2PC) — and why you should avoid it.

**Answer:**

2PC is the classical protocol for atomic transactions across multiple participants. It works on paper, fails in production. Modern systems prefer Saga + idempotency over 2PC almost everywhere.

### The Protocol

Coordinator + N participants.

**Phase 1 (prepare):**
```
coordinator → all participants: PREPARE
participants: do the work, lock resources, write to log
participants → coordinator: VOTE YES or NO
```

**Phase 2 (commit/abort):**
```
if all YES:
   coordinator → all: COMMIT
   participants: apply changes, release locks
else:
   coordinator → all: ABORT
   participants: roll back
```

Each participant is either committed or rolled back; never partial.

### Why It Works (In Theory)

Each participant logs its decision before voting. If anyone crashes, on recovery they consult their log and the coordinator to learn the final outcome.

### Why It's Bad (In Practice)

**1. Blocking on coordinator failure.**

If the coordinator dies after Phase 1 + some Phase 2 messages sent but not others, participants are left **holding locks** waiting for the coordinator to recover. Could be hours.

```
coordinator says PREPARE to A, B, C
all reply YES
coordinator crashes
A, B, C: locked, waiting for commit/abort
```

No reliable way to resolve without coordinator. Other transactions wait.

**2. Holding locks across a network.**

Participant locks rows during PREPARE. Lock duration = network round trip × 2. Cross-DC = hundreds of ms. Throughput plummets.

**3. Doesn't scale.**

Coordinator is a bottleneck; participants block on each other. Latency = max(slowest participant).

**4. Doesn't handle some failures.**

If a participant votes YES but crashes before receiving COMMIT, on recovery it must reconcile. Possible, but adds protocol complexity (3PC tried, mostly failed).

### Where 2PC Lives Today

- **XA transactions** in JDBC + JMS: classic 2PC pattern. Performance and operational nightmare; rarely used.
- **Internal distributed-DB protocols**: Spanner uses 2PC under the hood across shards (with Paxos for durability). Works because Google built it carefully and locks are short.
- **CockroachDB, YugabyteDB**: similar Spanner-like distributed-SQL approach.

If you find yourself implementing 2PC in app code, stop.

### Replacements

**1. Saga**.

Sequence of local transactions + compensating actions. Eventually consistent; no global lock. See [Saga Pattern](../patterns/saga.md).

**2. Transactional outbox**.

Atomic write to DB + outbox in one local transaction. Downstream catches up async. See [Transactional Outbox](../patterns/outbox.md).

**3. Distributed SQL**.

If you really need ACID across data, use a system designed for it (Spanner, CockroachDB) and let it handle 2PC internally.

**4. Single-writer per entity**.

If only one node writes to a given entity, no coordination is needed.

### Three-Phase Commit (3PC)

Adds a "PRE-COMMIT" phase between PREPARE and COMMIT. Reduces blocking risk; doesn't eliminate it in async networks. Theoretically interesting; not used in practice.

### When 2PC Is Acceptable

- Small number of participants (2–3).
- Fast network (LAN, same DC).
- Short critical section.
- Failure tolerance acceptable (you're OK with manual recovery if coordinator dies).

Even then, the design conversation should justify why no alternative works.

### Common Mistakes

| Mistake | Reality |
|---------|---------|
| Cross-microservice 2PC | Operational nightmare; use Saga |
| Long-running transactions with 2PC | Locks held for minutes; system grinds |
| Coordinator as a singleton | SPOF; if you must, replicate via consensus |
| Treating "exactly-once" as the goal | Aim for exactly-once *effect* via idempotency |

> [!NOTE]
> 2PC fails open in the wrong direction: when in doubt, participants block, not commit. That means *any* coordinator failure can cause an unbounded outage. Avoid it.

### Interview Follow-ups

- *"How does Spanner do distributed transactions?"* — 2PC within Spanner, but each participant is itself a Paxos group; coordinator failure tolerated by majority. TrueTime narrows the lock window.
- *"What's the Saga alternative?"* — Sequence of local transactions; each step has a compensation. No global locks; eventual consistency.
- *"What did Google's MegaStore / Spanner papers teach us?"* — That 2PC + Paxos + clock synchronization + small batches *can* scale, with massive engineering effort. Most teams don't have that bandwidth.
