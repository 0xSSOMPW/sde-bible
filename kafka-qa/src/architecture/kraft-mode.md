# Q: How does KRaft mode work, and what changed when ZooKeeper was removed?

**Answer:**

KRaft (KIP-500) replaces ZooKeeper with a built-in **Raft consensus** quorum running inside Kafka brokers. As of Kafka 3.5+, KRaft is the default and ZooKeeper mode is deprecated; Kafka 4.0 removes ZK entirely.

### Why Remove ZooKeeper

ZK-based Kafka had two distinct systems with different operational profiles:

- **ZK** stored cluster metadata (topics, partitions, ACLs, configs).
- **Kafka brokers** stored data.

Pain points:
- Two systems to deploy, monitor, secure, upgrade.
- Metadata changes went through ZK's notification model — slow for many partitions.
- Controller fail-over was O(N) in topic count (had to reload every partition state).
- ZK's write throughput capped the maximum partition count at ~200k cluster-wide.

KRaft replaces this with a Raft log of metadata events, kept inside Kafka itself.

### Architecture

```
ZooKeeper mode:                    KRaft mode:

  ┌──────────┐                     ┌────────────────────────┐
  │  ZK      │                     │  Controllers (Raft)    │
  │ ensemble │                     │  c1   c2   c3          │
  └────┬─────┘                     └──────────┬─────────────┘
       │ metadata                              │ metadata via Raft log
       │                                       │
  ┌────┴───────┐                     ┌────────┴─────────────┐
  │  Brokers   │                     │  Brokers             │
  │ b1 b2 b3   │                     │  b1   b2   b3        │
  └────────────┘                     └──────────────────────┘
```

- **Controller quorum**: typically 3 (or 5) nodes that own metadata.
- **Brokers**: handle data, replicate from each other, fetch metadata from the controller quorum.
- Same process can run as `process.roles=broker,controller` (combined mode, for small clusters) or split (recommended for production).

### The Metadata Log

Cluster metadata is stored on a special internal topic: `__cluster_metadata`. Single partition. Replicated across all controller nodes via Raft.

Every metadata change is an **event** in this log:
- Create topic → `TopicRecord`.
- Change partition leader → `PartitionChangeRecord`.
- Register broker → `RegisterBrokerRecord`.
- Update ACL → `AccessControlEntryRecord`.

Each broker replays the log into an in-memory image. On startup, brokers consume from offset 0 (or from a snapshot) and end at the high-water mark.

### Snapshots

Replaying the entire log on every restart would be slow on big clusters. KRaft periodically writes a **snapshot** of the current metadata image:

```
__cluster_metadata segments:
  00000000000000000000.log
  00000000000000000000.checkpoint           <-- snapshot
  00000000000019234567.log
```

New brokers load the snapshot, then replay only the delta. Startup goes from minutes (with ZK) to seconds.

### Configuration

`server.properties`:

```properties
process.roles=broker,controller             # combined; or just broker / controller
node.id=1
controller.quorum.voters=1@c1:9093,2@c2:9093,3@c3:9093
listeners=PLAINTEXT://:9092,CONTROLLER://:9093
inter.broker.listener.name=PLAINTEXT
controller.listener.names=CONTROLLER
log.dirs=/var/kafka/data
metadata.log.dir=/var/kafka/metadata
```

Bootstrap a new cluster:

```bash
kafka-storage.sh random-uuid
# returns cluster id, e.g. abc123...

kafka-storage.sh format -t abc123... -c server.properties
# initializes log dirs

kafka-server-start.sh server.properties
```

### Operational Differences

| Aspect | ZK | KRaft |
|--------|----|-------|
| Components to deploy | 2 (Kafka + ZK ensemble) | 1 (Kafka only) |
| Metadata storage | ZK's znodes | `__cluster_metadata` topic |
| Controller fail-over | Slow (reload state) | Fast (in-memory image hot) |
| Max partitions | ~200k cluster-wide | Millions tested |
| ACL store | ZK | KRaft metadata |
| `kafka-acls.sh` admin | Talks to ZK | Talks to broker via `--bootstrap-server` |
| `kafka-configs.sh` | Talks to ZK | Talks to broker |
| Upgrade path | ZK upgrade separately | Single rolling restart |

### Combined vs Isolated Mode

**Combined** (`process.roles=broker,controller`):
- Same node is both. Saves machines.
- Production-supported for small clusters (≤ 5 nodes).
- Failure of one node loses both broker capacity and controller voter.

**Isolated** (separate broker and controller nodes):
- Recommended for production at scale.
- Controllers are small (a few hundred MB RAM, low CPU) — cheap nodes.
- Brokers don't share resources with consensus traffic.
- Independent scaling.

### Migration from ZK to KRaft

Kafka 3.4+ supports a documented migration:

1. Upgrade cluster to a KRaft-capable Kafka version.
2. Provision a KRaft controller quorum.
3. Set `zookeeper.metadata.migration.enable=true` on brokers and controllers.
4. Wait for migration to complete (metadata copied from ZK to KRaft log).
5. Roll brokers to remove ZK config.
6. Decommission ZK.

Non-trivial; test in non-prod first.

### What Changed for Operators

- **Admin CLIs** now require `--bootstrap-server <broker>` not `--zookeeper <zk>`. Older scripts break.
- **Monitoring**: ZK-specific metrics gone. New KRaft metrics: `kafka.controller:type=KafkaController,name=...`.
- **Backup**: ZK snapshots no longer relevant. Back up `metadata.log.dir` from controllers (or rely on Raft replication).
- **DNS / connection strings**: clients only ever needed broker addresses (ZK was admin-only).

### KRaft Failure Modes

- **Lose minority of controllers**: cluster keeps operating; voter quorum still met.
- **Lose majority of controllers**: metadata operations halt (Raft can't make progress). Brokers keep serving cached metadata; producers/consumers continue until they need a metadata update.
- **Disagreement** (split brain): impossible — Raft ensures one leader at a time.

### Common Mistakes

| Mistake | Fix |
|---------|-----|
| Even number of controllers | Use odd (3, 5) — quorum math |
| Combined mode for a 50-node cluster | Use isolated controllers |
| Sharing `metadata.log.dir` with data dirs | Put metadata on its own (small) fast disk |
| Forgetting to back up controllers separately | Snapshots + Raft replication; document the recovery path |
| Old admin scripts still referencing `--zookeeper` | Update everything to `--bootstrap-server` |

> [!NOTE]
> KRaft is not a feature you "enable" — it's the new default mode of running Kafka. Treat ZK mode as legacy and plan migrations within the support window.

### Interview Follow-ups

- *"Why Raft over Paxos?"* — Raft's understandability and existing tooling. Plus Kafka already had append-only log semantics — a natural fit.
- *"What's the controller leader election?"* — Raft leader election. One controller is the active leader; others are followers/voters.
- *"Can KRaft be used standalone like ZK was?"* — No, it's embedded in Kafka. Other systems that needed ZK (HBase, Solr) are not affected by Kafka's KRaft move.
