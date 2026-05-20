# Q: Consensus — Raft and Paxos. What do they solve, how do they work?

**Answer:**

Consensus protocols let a cluster of nodes **agree on a value** despite failures and message delays. They're the foundation of every distributed system that claims correctness: etcd, Consul, ZooKeeper, Spanner, CockroachDB, Kafka KRaft.

### The Problem

Multiple servers must agree on:
- Who is the leader.
- The order of operations in a replicated log.
- Membership of the cluster.

Despite:
- Crashes (fail-stop).
- Network partitions.
- Message reordering, delay.

But not despite:
- Byzantine failures (servers lying) — outside scope; need different protocols.

### Properties

A correct consensus protocol guarantees:

- **Safety**: agree on the same value (no two correct nodes decide differently).
- **Liveness**: eventually decide a value (assuming a majority is up).

Famous result: **FLP impossibility** — no deterministic protocol guarantees both safety and liveness in a fully async network with even one crash. In practice, we use timeouts and accept brief liveness violations during failures.

### Raft (Understandable)

Raft splits consensus into three sub-problems:
1. Leader election.
2. Log replication.
3. Safety (rules ensure only safe logs win elections).

#### Election

Each node is `Follower`, `Candidate`, or `Leader`.

```
Follower: hears heartbeats from leader; if timeout, become Candidate.

Candidate:
  ++current_term
  vote for self
  request votes from peers

Receive majority of votes? → become Leader.
Receive heartbeat with higher term? → become Follower.
Election timeout? → start new election.
```

Election timeout randomized to prevent split votes (e.g., 150–300 ms).

#### Log Replication

Leader receives client requests, appends to its log, replicates to followers.

```
client → leader.append(cmd)
leader: append to its log at index i
leader: send AppendEntries to all followers
followers: write to log; reply OK
leader: when majority confirmed → mark index i committed
leader: apply to state machine; respond to client
leader: tell followers of new commit index on next heartbeat
```

Once committed, the entry is durable — survives leader failure.

#### Safety

Rules ensure no leader is elected without all committed entries:

- Vote denied to candidate whose log is shorter / older.
- Leader never overwrites committed entries.
- Log matching property: same index + term → same prefix.

### Paxos

Older, more general, harder to understand. Multi-Paxos for replicated logs.

Two phases per decision:
1. **Prepare**: proposer picks a number n, asks acceptors to promise not to accept lower n.
2. **Accept**: with a majority of promises, proposer asks acceptors to accept its value.

Leader-based optimizations (Multi-Paxos) skip Phase 1 once a leader is stable — looks similar to Raft in practice.

Paxos paper is notoriously dense. Raft was designed (and named) for understandability.

### Raft vs Paxos

| Aspect | Raft | Paxos |
|--------|------|-------|
| Mental model | Leader + log + elections | Proposers + acceptors + rounds |
| Pedagogy | Clear | Difficult |
| Implementations | etcd, Consul, Kafka KRaft, TiKV, CockroachDB | Chubby, Spanner (some variants), ZooKeeper-Zab (similar) |
| Performance | Comparable to Multi-Paxos | Comparable |
| Variants | Joint consensus, Membership change | Multi-Paxos, Cheap Paxos, Fast Paxos |

Practical guidance: **use Raft** unless you have a specific reason for Paxos. Most modern systems do.

### Where Consensus Is Used

| System | Why |
|--------|-----|
| etcd, ZooKeeper, Consul | Distributed coordination (locks, leader election, config) |
| Kafka KRaft | Cluster metadata (formerly via ZooKeeper) |
| CockroachDB, Spanner, TiKV | Per-range / per-partition consensus for ACID |
| MongoDB replica set | Modified Raft for primary election |
| Hashicorp products (Nomad, Vault) | Built on Raft |

### Quorum

Most operations need a **majority** of nodes (`floor(N/2) + 1`).

- 3 nodes → 2 quorum → tolerate 1 failure.
- 5 nodes → 3 quorum → tolerate 2 failures.
- 7 nodes → 4 quorum → tolerate 3 failures.

Larger N tolerates more failures but writes slow (must wait for majority).

Best practice: 3 or 5. Odd numbers avoid split-brain ties.

### Failure Modes

| Failure | Result |
|---------|--------|
| Minority dies | Cluster keeps writing (majority intact) |
| Majority dies | Cluster can't make progress; writes halt until restoration |
| Network partition (split) | Majority side runs; minority is read-only or stops |
| Leader fails | Election picks new leader (~ election timeout latency) |
| Slow leader | Holds back throughput; followers eventually trigger election |

### Performance

- Write latency = 1 RTT to quorum (best case).
- Read latency: from leader (cheapest); from follower (eventual).
- Throughput: bounded by leader's I/O.

Scale beyond one Raft group: **shard the keyspace**, one Raft group per shard (Spanner, CockroachDB do this).

### Pitfalls

- **Split brain after partition heals**: the protocol prevents it, but only if implemented correctly.
- **Slow consensus over WAN**: round-trip dominates; consider regional clusters.
- **Disk fsync stalls**: AppendEntries requires fsync; cheap drives kill throughput.
- **Membership changes**: must use joint consensus or single-node-at-a-time, not raw replacement.

### Common Mistakes

| Mistake | Reality |
|---------|---------|
| Using consensus for everything | Way overkill; only for metadata / leader-coordinated state |
| 2-node "quorum" | Worst of both: needs both up → less availability than 1 |
| Even number of nodes | Same fault tolerance as N-1 odd; waste a node |
| Ignoring election timing | Too short → flaps; too long → outages |

> [!NOTE]
> Consensus is the gold-plated screw of distributed systems. Use it for the small piece of state that *must* be linearizable across nodes. Build the rest on cheaper primitives (gossip, eventual consistency).

### Interview Follow-ups

- *"How does Raft handle leader failure?"* — Election timeout triggers; candidates emerge; majority elects new leader; minimum disruption to clients (retries on new leader).
- *"What is joint consensus?"* — Raft's safe membership-change protocol: both old and new configurations active during transition.
- *"Difference between Raft and Multi-Paxos?"* — In practice, similar performance. Raft has stricter leader behavior; Paxos allows proposals from any node. Raft easier to implement correctly.
