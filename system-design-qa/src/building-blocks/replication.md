# Q: Replication topologies — single-leader, multi-leader, leaderless. Trade-offs.

**Answer:**

Replication maintains copies of data on multiple machines for **availability**, **durability**, and **read scaling**. The three topologies — single-leader, multi-leader, leaderless — each pick a different point in the consistency / availability / latency space.

### Single-Leader (Primary-Replica)

```
        writes
client ───────► leader ───────► follower
                  │  ───────► follower
                  │
                async or sync replication
```

- All writes go to one leader.
- Followers receive a stream of changes (binlog, WAL).
- Reads can go to followers (eventual) or leader (strong).

Used by: Postgres, MySQL, MongoDB (replica set), most managed RDBMS.

**Sync vs async replication:**

- **Async**: leader acks write before followers apply. Fast. **Risks data loss** if leader dies before replication.
- **Sync to one**: at least one follower must ack. Safer, slower.
- **Sync to majority** (Raft-style): durable but cross-AZ latency tax.

Failover:
- Detect leader failure.
- Promote most up-to-date follower.
- Reroute clients.

Window where writes are lost = unreplicated lag at moment of crash.

### Multi-Leader (Active-Active)

```
       writes
client ───► leader A  ◄───► leader B  ◄─── client (other region)
                ▲                ▲
                └───── replicate ─┘
```

Multiple leaders, often one per geographic region. Each accepts writes locally; replicates async to others.

Used by: BDR (Postgres), CouchDB, multi-region MySQL with custom plumbing, Cassandra (per-DC), Cosmos DB multi-write.

**Trade**: local-region latency, but **write conflicts** between leaders. Same key updated in A and B concurrently → which wins?

Conflict resolution:
- **Last-write-wins (LWW)**: timestamp-based; can drop data.
- **App-defined merge**: e.g., union for sets, max for counters.
- **CRDTs** (G-Counter, OR-Set): merge correctly by construction.
- **Manual**: store both, raise to user (rare).

### Leaderless

```
                  ┌─► node A
client ──────────►├─► node B
                  └─► node C
```

Writes go to **multiple nodes** in parallel. Reads also go to multiple nodes and the client (or coordinator) picks the latest version (often via timestamps).

Used by: Cassandra (default), DynamoDB, Riak, ScyllaDB.

**Quorum math** (`N` replicas, `W` write acks needed, `R` read acks needed):

```
W + R > N    → strong consistency for that op
W + R ≤ N    → eventual
```

Common: `N=3, W=2, R=2` (read/write quorum).

**Read repair / anti-entropy**:
- Read-time repair: when read returns mismatched versions, write the latest back to lagging replicas.
- Merkle-tree-based background sync.
- Hinted handoff: temporarily store writes for an unreachable replica.

### Sync vs Async Tradeoffs

| Mode | Latency | Durability | Availability |
|------|---------|-----------|--------------|
| Single async | Lowest | Lose recent writes on failover | High (no quorum needed) |
| Single sync to one | Low | Survive 1 failure | Medium |
| Single sync to majority (Raft) | Medium | Survive minority failure | High |
| Multi-leader | Local-fast | Conflict-prone | High |
| Leaderless quorum | Configurable per op | Configurable | Highest |

### Replication Lag

For async or leaderless reads, the follower can be **behind**. Implications:

```
You write to leader: balance = 100
Immediately read from follower: balance = 90 (lag of 1 second)
```

Patterns to handle:

- **Read-your-writes**: route reads to the leader for the user's recent writes (sticky for a window).
- **Monotonic reads**: pin the user to one follower so reads don't bounce between replicas at different lags.
- **Bounded staleness**: SLA on max lag; alert if exceeded.

### Failover and Split Brain

When a leader is unreachable, who promotes?

- **Manual**: ops team flips the switch. Slow but safe.
- **Automated with consensus**: Raft/Paxos for promotion. Used by etcd, Consul, modern Postgres (Patroni).
- **STONITH** (Shoot The Other Node In The Head): kill the old leader to prevent two-leader scenarios.

Split brain: network partition makes each side think it's the leader. Both accept writes → divergence. Mitigations:
- Require quorum to be leader.
- Fence the minority side via STONITH.
- Use a coordination service (etcd) for leases.

### Geographic Replication

For multi-region, choose:

| Pattern | Trade |
|---------|-------|
| Single leader, async to other regions | Cheap, lose data on region loss |
| Single leader, sync to other region | Cross-region write latency (~80–150 ms) |
| Multi-leader per region | Local writes fast, conflict resolution required |
| Distributed consensus (Spanner) | Linearizable across regions; expensive |

### Replicated Logs

The underlying mechanism for most replication is a **replicated log** (WAL, binlog, Raft log, Kafka). Followers consume the log and apply changes in order.

Why logs:
- Order is unambiguous.
- Replay is idempotent (LSN-based).
- Lag is observable (offset distance).
- Recovery = "catch up the log."

### Backups vs Replicas

Replicas are not backups. Both protect against hardware failure. Only backups protect against logical errors (bad query deleted everything; replica deleted it too).

Standard: replicas for availability, snapshots + point-in-time recovery for logical disasters.

### Common Mistakes

| Mistake | Reality |
|---------|---------|
| Treating replicas as backup | A `DROP TABLE` propagates everywhere |
| Reading from replica when read-your-writes needed | Stale data, confused users |
| No alerting on replication lag | Lag balloons unnoticed; failover loses data |
| Manual failover with no documented runbook | Outage longer than necessary |
| Multi-leader with last-write-wins on financial data | Silent data loss |

> [!NOTE]
> Pick a replication topology by the question "what should happen during a partition?" Single-leader CP loses availability. Leaderless AP loses recency. Multi-leader trades both for latency. There's no free combination.

### Interview Follow-ups

- *"How would you do zero-downtime failover?"* — Streaming replication + auto-promotion + connection draining + retry on the client side.
- *"What is semi-sync replication?"* — Leader waits for at least one follower's ACK before responding to client. Bounded data-loss window.
- *"When is multi-leader the right call?"* — Multi-region active-active where local-write latency matters AND your data tolerates merge / LWW (calendars, document collaboration, social).
