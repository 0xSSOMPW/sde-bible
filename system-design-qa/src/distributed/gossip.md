# Q: Gossip protocols — what they are, where they're used.

**Answer:**

Gossip (epidemic) protocols spread state through a cluster by **periodic peer-to-peer exchanges**. They're decentralized, fault-tolerant, and scale to thousands of nodes — but eventually consistent.

### How It Works

```
every second, each node:
  pick K random peers
  exchange state digests
  update local view with newer entries
```

Information spreads exponentially: O(log N) rounds to reach all nodes.

```
round 0:  1 node knows
round 1:  K nodes know
round 2:  K² nodes
...
round log_K(N): all nodes know
```

For K=3, N=1000: ~7 rounds.

### Properties

- **Decentralized**: no leader.
- **Fault-tolerant**: lose any node; gossip continues.
- **Scalable**: each node talks to constant fanout regardless of cluster size.
- **Eventually consistent**: not linearizable.
- **Probabilistic delivery**: with overwhelming likelihood.

### Use Cases

| System | What it gossips |
|--------|----------------|
| Cassandra | Cluster membership, token ranges, schema |
| Redis Cluster | Cluster topology, slot ownership |
| Consul | Node health (Serf library) |
| HashiCorp Nomad | Node + job status |
| Akka Cluster | Cluster state |
| Bitcoin / Ethereum | Transaction + block propagation |
| Dynamo / Riak | Membership |

### Anti-Entropy

The reconciliation step when two nodes exchange data and find disagreement:

```
Node A: { keyX: v3 }
Node B: { keyX: v2 }

A → B: "I have keyX = v3"
B: updates to v3
```

For large state, exchanging hashes (Merkle trees) reduces traffic. Cassandra uses this for SSTable repair.

### SWIM Protocol

Specialized membership gossip (used by Consul, HashiCorp Serf):

1. **Failure detection**: probe random peer; if no reply, ask other peers to probe.
2. **Dissemination**: piggyback membership events on probes.

SWIM gives bounded failure-detection time + scalable.

### Push vs Pull vs Push-Pull

- **Push**: I send my state to a peer. Effective when info is new.
- **Pull**: I ask the peer for theirs. Effective when info is established.
- **Push-pull**: both directions in one exchange. Most common.

### Convergence Time

With K=3 peers per round and 1-second interval:
- 100 nodes: ~5 seconds to fully converge.
- 1000 nodes: ~7 seconds.
- 10000 nodes: ~9 seconds.

The "logarithmic-in-N" property is what makes gossip suitable for large clusters.

### Failure Modes

| Failure | Result |
|---------|--------|
| Network partition | Two halves diverge until partition heals; converge on heal |
| Node restart | Picks up state by gossiping with peers |
| Slow node | Information takes longer to reach it; cluster continues |
| Byzantine node | Gossip protocols don't tolerate this; need different protocol |

### Anti-Entropy vs Rumor-Mongering

Two modes:
- **Anti-entropy**: continuously reconcile entire state. Bandwidth-heavy; always converges.
- **Rumor-mongering**: spread only new info; stop after seeing it sufficient times. Bandwidth-light; small chance of missing.

Production systems mix: rumor for events, periodic anti-entropy for safety.

### Limitations

- **Eventually consistent**: never use for "must be ordered" decisions.
- **Bandwidth**: chatty on big clusters; tune intervals.
- **No durability**: lost on restart unless persisted separately.
- **Byzantine tolerance**: only via stronger protocols (PBFT etc.).

### Common Mistakes

| Mistake | Reality |
|---------|---------|
| Using gossip for transactions | Doesn't give consistency; use Raft |
| No version on gossiped messages | Can't detect newest; use vector clocks or timestamps |
| Too frequent gossip | Bandwidth spike; tune interval |
| Forgetting fanout / lack of bound | Gossip messages exponentially balloon if not gated |

> [!NOTE]
> Gossip is the right tool for **soft state**: "who's in the cluster," "what's the cluster's view of itself," "node health." It's the wrong tool for any decision that must be exact.

### Interview Follow-ups

- *"Why does Cassandra use gossip?"* — Membership + topology shared across all nodes; node count can be hundreds; no central coordinator desired.
- *"Difference between gossip and pub-sub?"* — Pub-sub: broker-centric, push only, often ordered. Gossip: peer-to-peer, eventual, no broker.
- *"How fast can gossip detect a node failure?"* — Configurable via SWIM probe intervals; typically 5–30 seconds depending on tuning.
