# Q: How do you elect a leader in a distributed system?

**Answer:**

Many systems need exactly one node to perform a role (write coordinator, scheduler, cron). **Leader election** picks that one node and re-picks when it dies. The wrong implementation produces split-brain; the right one uses a consensus-backed lease.

### Why You Need a Leader

- Single writer (avoids conflict resolution).
- Coordinator for distributed work (sharding, locking).
- Reduce thundering herd (one node decides, others follow).

Examples: Kafka controller, Raft leader, Kubernetes controller-manager, scheduled-job runner.

### Naive Approaches (Don't)

**Lowest IP** wins. Works until two nodes disagree about who's "up."

**Random**: two could pick simultaneously.

**Acquire a row lock in DB**: lock holder is leader. Sort of works; DB outage = no leader.

These have **split-brain** risk: two leaders run simultaneously, both perform "exclusive" work, data corrupts.

### The Right Pattern: Lease + Consensus

```
Coordinator (etcd / ZooKeeper / Consul)
   acquire-lease(key="leader", ttl=30s)
      if no one holds it: grant
      else: deny

Leader periodically renews the lease (e.g., every 10s, before TTL expiry).
If leader fails to renew → lease expires → another candidate acquires.
```

The coordinator runs Raft/Paxos internally; the lease grant is linearizable.

### etcd Example

```python
lease = etcd.lease(ttl=30)
ok = etcd.put("leader", "node-1", lease=lease.id, prevExist=False)
if ok:
    print("I am leader")
    while running:
        lease.keepalive()
else:
    watch("leader")     # become leader when current expires
```

### ZooKeeper Pattern

Ephemeral sequential nodes:

```
each candidate creates /election/node_xxxx (auto-numbered, ephemeral)
the one with the lowest number is the leader
candidates watch the node ahead of them
if that node dies (session closes → ephemeral disappears) → next one becomes leader
```

ZooKeeper sessions ensure failed nodes are reliably detected.

### Kubernetes Lease Object

The standard way in K8s:

```yaml
apiVersion: coordination.k8s.io/v1
kind: Lease
metadata:
  name: my-controller
spec:
  holderIdentity: pod-1
  leaseDurationSeconds: 15
  renewTime: "..."
```

Controllers compete by updating the Lease object. K8s API server (backed by etcd) gives linearizable updates.

Built into `client-go` and operator SDKs.

### Common Mistakes

**Split-brain via short network blip.**

Leader's link blips, lease expires, follower takes over. Original leader's network recovers — it still thinks it's leader. Now two leaders.

Mitigation: leader checks its lease before each operation; abdicates if lease has expired or been taken. **Fencing tokens** (see below) make this bulletproof.

### Fencing Tokens

Each lease grant comes with a monotonic counter. Downstream systems reject writes from older fence tokens:

```
acquire-lease → grant with fence=42
   → write to DB with fence=42

later:
acquire-lease → grant with fence=43
   → write to DB with fence=43

old leader (still thinks it's leader) tries to write with fence=42
   → DB rejects (fence < current)
```

Required for correctness when downstream systems matter (DBs, queues).

Implemented in: etcd revision numbers, ZooKeeper zxid, Kafka epochs.

### When Leader Election Fails

- **No consensus possible** (majority of coordinator nodes down): no leader; system degrades.
- **Repeated elections (flapping)**: usually network instability; increase lease TTL.
- **Two leaders simultaneously**: bug. Add fencing tokens, verify lease before critical operations.

### Performance

- Lease renewal cost: 1 round trip every N seconds (cheap).
- Election after failure: lease TTL + acquire time (~seconds).
- Acceptable for control-plane work; not for hot path.

### Use Cases by Frequency

- **High-frequency coordination** (e.g., per-request leader): wrong tool; restructure.
- **Per-job leader** (cron, distributed scheduler): yes.
- **Per-partition leader** (Kafka): each partition has its own.
- **Cluster-wide leader** (k8s controller, Kafka controller): yes.

### Trade-offs

**Long TTL**: faster failover, lower coordinator load, longer outages.

**Short TTL**: quicker failure detection, higher coordinator chatter, brief flap windows.

Typical: 15-30 second TTL with 5-second renewal.

### Common Mistakes

| Mistake | Fix |
|---------|-----|
| Self-rolled leader election | Use etcd/ZK; correctness is hard |
| No fencing token at downstream | Two leaders write simultaneously |
| Renewing only on heartbeats with long network gap | Lease expires before renewal arrives; increase margin |
| Multiple unrelated services sharing one leader election | Per-service lease — finer-grained |

> [!NOTE]
> The lesson of every distributed-systems incident report: trust your lease + use fencing tokens. Without them, "we thought we had one leader" is the punchline.

### Interview Follow-ups

- *"How does Kafka elect the controller?"* — In KRaft, Raft itself among the controller quorum. In ZK mode, candidates registered ephemeral znode; first wins; reelection on failure.
- *"What's bully algorithm?"* — Classic textbook leader election by node ID. Educational; not used in production (no fencing, sensitive to clock skew).
- *"Why not just use a DB row lock?"* — Works for simple cases; doesn't handle the DB's own failover, can't fence downstream operations.
