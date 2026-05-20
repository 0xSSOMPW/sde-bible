# Q: What are CAP and PACELC, and how do they shape distributed-system design?

**Answer:**

CAP is a forced-choice theorem about distributed databases under network failure. PACELC extends it to the *non-failure* case, where latency vs consistency is the real trade. Together they describe almost every storage system on the market.

### CAP Theorem (Brewer, 2000)

In the presence of a **network partition**, a system can provide **at most two of**:

- **C — Consistency**: every read sees the most recent committed write (linearizability).
- **A — Availability**: every request gets a non-error response.
- **P — Partition tolerance**: system keeps working despite dropped/delayed messages.

Critical: **P is not optional** in any real distributed system. Networks fail. So the actual choice is **CP vs AP** during a partition.

```
                     Partition occurs
                          │
            ┌─────────────┴──────────────┐
            ▼                            ▼
        CP system                    AP system
   refuse some requests          accept all requests
   (return errors)               (may return stale data)
   guarantees consistency        guarantees availability
```

### Example Classifications

| System | Class | Behavior under partition |
|--------|-------|--------------------------|
| Spanner, CockroachDB | CP | Refuses writes that can't reach majority |
| ZooKeeper, etcd | CP | Loses minority side |
| DynamoDB (eventual) | AP | Both sides accept writes, reconcile later |
| Cassandra (default) | AP | Tunable per query via consistency level |
| Postgres (single primary) | CP | Standby read-only if primary unreachable |
| MongoDB | CP (default) | Minority partition rejects writes |
| Riak | AP | Sloppy quorum, hinted handoff |

### PACELC (Abadi, 2012)

CAP only describes partition behavior. PACELC adds the steady-state question:

> **If Partition: choose A or C. Else (no partition): choose L (Latency) or C (Consistency).**

A system is described by both halves: `PA/EL` (DynamoDB), `PC/EC` (Spanner), `PA/EC` (MongoDB sometimes), `PC/EL` (some configs).

The "EL or EC" axis matters because **synchronous replication** to remote replicas adds latency. Choosing EC means waiting for ack from N regions before responding to the client.

### Tunable Consistency

Modern systems rarely pick one corner. They expose **per-operation** knobs:

```sql
-- Postgres: synchronous_commit per transaction
SET LOCAL synchronous_commit = on;        -- wait for replica fsync

-- Cassandra
SELECT ... USING CONSISTENCY QUORUM;       -- per query

-- DynamoDB
GetItem(ConsistentRead=true)               -- per operation
```

You buy consistency only where you need it. Eventual reads for product browse; strong reads for checkout.

### Quorum Math

For N replicas:

```
W + R > N   →  strong consistency on reads
W + R ≤ N   →  eventually consistent
```

Common configs:
- `N=3, W=2, R=2`: tolerates 1 node loss, strong reads via quorum.
- `N=3, W=3, R=1`: writes block on full ISR; reads are fast.
- `N=3, W=1, R=1`: AP — fast, eventually consistent.

### What CAP Is NOT

- Not about whether one node is down. Single-node failures are a different problem.
- Not a static label for an entire system. Same DB can run CP or AP depending on config.
- Not a green light to pick "any two." You always need P.

### Interview Use

When asked "what's the consistency model of your design?":

1. State the failure mode you're optimizing for.
2. Choose CP or AP **per data class**, not for the whole system. Orders → CP. Likes → AP.
3. Mention quorum settings, replication topology, and what happens during partition (errors vs stale reads).

### Common Mistakes

| Mistake | Reality |
|---------|---------|
| "Cassandra is AP, full stop" | Tunable per query: `CONSISTENCY ONE` (AP) vs `ALL` (CP-leaning) |
| "We can have CA — we're inside one AZ" | Cross-rack network partitions still happen |
| "Spanner beats CAP with TrueTime" | It's still CP — just smaller partition windows |
| Using CAP to defend eventual consistency for a bank balance | Pick the right axis per data domain |

> [!NOTE]
> CAP is the *partition-time* trade-off. PACELC is the *steady-state* trade-off. Real systems live mostly in steady state — PACELC matters more day-to-day, but CAP determines what happens when the network actually breaks.

### Interview Follow-ups

- *"How would you build a CP system on top of an AP store?"* — Wrap with a consensus layer (Raft) above the keyspace; trade availability for linearizability.
- *"Why doesn't Spanner break CAP?"* — It doesn't. It's CP. TrueTime narrows the wait window to ~7 ms, making the consistency cost cheap.
- *"What's linearizability vs serializability?"* — Linearizability is about *single-object* real-time ordering. Serializability is about *transactions* being equivalent to some serial order. SQL DBs offer the latter; distributed KVs argue about the former.
