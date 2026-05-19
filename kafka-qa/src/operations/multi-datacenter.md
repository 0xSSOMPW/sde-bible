# Q: How do you replicate Kafka across data centers — MirrorMaker 2 vs Cluster Linking?

**Answer:**

A single Kafka cluster lives in one fault domain. To survive a region outage, support active-active multi-region traffic, or aggregate edge clusters into a central one, you need **inter-cluster replication**. Two main tools: open-source **MirrorMaker 2** and Confluent's proprietary **Cluster Linking**.

### Why You Replicate

Common drivers:
- **Disaster recovery (DR)**: failover to a standby cluster if primary region dies.
- **Aggregation**: many edge clusters → one central analytics cluster.
- **Geo-distribution**: producers/consumers near their users; mirror for cross-region reads.
- **Migration**: move workloads between clusters with no downtime.
- **Compliance**: pin a copy of data in a specific region.

### MirrorMaker 2 (MM2)

A Kafka Connect–based replicator (replaces MM1). Runs as a fleet of Connect workers.

```
   ┌──────────────┐                          ┌──────────────┐
   │  Cluster A   │  ───  MirrorMaker 2  ──► │  Cluster B   │
   │  topic foo   │                          │  topic A.foo │
   └──────────────┘                          └──────────────┘
```

Topics are **renamed by default** — `foo` on A becomes `A.foo` on B — to make active-active safe (no infinite loops). Configurable via `replication.policy`.

```properties
# mm2.properties
clusters = primary, dr
primary.bootstrap.servers = primary:9092
dr.bootstrap.servers = dr:9092

primary->dr.enabled = true
primary->dr.topics = orders|payments|inventory
primary->dr.replication.factor = 3

replication.policy.separator = .
sync.topic.acls.enabled = true
sync.topic.configs.enabled = true

offset-syncs.topic.replication.factor = 3
```

MM2 also:
- Mirrors **topic configs** (retention, partition count delta).
- Mirrors **ACLs** (optional).
- Translates **consumer offsets** so a consumer can resume on the DR cluster (using `RemoteClusterUtils.translateOffsets`).

### Cluster Linking (Confluent)

Native broker-side replication — no separate Connect workers. The destination cluster pulls bytes directly from the source.

```
   ┌──────────────┐                          ┌──────────────┐
   │  Cluster A   │  ◄── byte-for-byte ───   │  Cluster B   │
   │  topic foo   │                          │  topic foo   │
   └──────────────┘                          └──────────────┘
```

Properties:
- **No topic renaming.** Same topic name on both sides (one-way).
- **Same offsets.** Consumers can fail over without offset translation.
- **Lower latency, fewer hops.** Brokers fetch directly.
- Confluent Platform / Cloud only.

### Comparison

| Aspect | MirrorMaker 2 | Cluster Linking |
|--------|---------------|-----------------|
| Open source | Yes | No (Confluent) |
| Extra components | Connect cluster | None |
| Topic name | Renamed (`source.topic`) | Same |
| Offsets | Translated via tool | Identical |
| Cross-cloud, cross-vendor | Yes | Confluent-only |
| Throughput overhead | Re-produces records | Byte-for-byte copy (cheaper) |
| Setup complexity | Connect ops to learn | One CLI call |

### Active-Passive (DR) Topology

```
Region A (active)               Region B (standby)
┌───────────────┐               ┌───────────────┐
│  Producers ───┼──┐    MM2     │   (idle)      │
│  Consumers ───┘  │ ─────────► │  Consumers    │
└───────────────┘                └───────────────┘
                                  ▲
                                  └─ used on failover
```

On failover:
1. Stop producers in A.
2. Wait for MM2 to drain.
3. Translate consumer offsets to DR cluster.
4. Point producers + consumers at B.

RPO (recovery point objective) ≈ replication lag, typically seconds.

### Active-Active Topology

```
Region A                      Region B
┌──────────┐                  ┌──────────┐
│   foo    │ ──── MM2 ──────► │ A.foo    │   <- reads from both
│ A.foo  ◄─┼────── MM2 ───────┤   foo    │
└──────────┘                  └──────────┘
```

Producers write to local `foo`. Consumers subscribe to both `foo` and `A.foo` (or `B.foo`).

Renaming prevents loops: `A.foo` on B is not re-replicated to A (MM2's default policy skips already-replicated topics).

Caveats:
- Consumer applications must merge two streams. Per-key ordering across regions is impossible without conflict resolution.
- Use case fits **independent streams** (per-region orders) better than **shared state** (global inventory).

### Aggregation Topology

```
edge1 ┐
edge2 ┼─► MM2 ───► central
edge3 ┘
```

Many small clusters → one big cluster for analytics. Each edge ships its local topic to a region-prefixed topic centrally (`edge1.orders`, `edge2.orders`...). Central jobs consume all of them.

### Operational Concerns

**Lag monitoring**:

```
mm2-MirrorSourceConnector.records.lag
mm2-MirrorCheckpointConnector.offset.lag
```

Alert if lag grows beyond a few seconds of replication.

**Throughput**:

MM2 scales horizontally — add Connect workers. Each task handles a set of partitions. Tune `tasks.max` and `producer.linger.ms` for throughput vs latency.

**Cycle prevention**:

`DefaultReplicationPolicy` rejects topics already named with a source prefix. If you use a custom policy, be sure to encode "this came from cluster X, don't bounce back."

**Schema sync**:

Schemas (Avro, Protobuf) must also be replicated if you use Schema Registry. Use a *separate* Schema Registry per cluster or a federated setup; otherwise consumers in DR can't deserialize.

### Common Mistakes

| Mistake | Fix |
|---------|-----|
| Forgetting to mirror Schema Registry | Consumers in DR can't deserialize |
| MM2 running on the destination cluster's network only | Run close to *source* to amortize wide-area cost |
| Not testing failover | RPO/RTO numbers on paper, broken in practice |
| Cross-region replication of compacted topics with same key in both | Conflict, last-writer-wins — design schema accordingly |
| ACL drift between clusters | Enable `sync.topic.acls.enabled` |

### Network Cost

Inter-region bandwidth is expensive. Estimate:

```
bytes/sec = ingress_to_replicate × compression_ratio
$/month  = bytes/sec × seconds × cloud_egress_rate
```

For a 100 MB/s replicated stream cross-region at $0.02/GB egress: ~$5k/month. Compression matters — `lz4`/`zstd` cuts cost ~3x.

### Migration Pattern: Cluster Linking Promotion

For zero-downtime migration:
1. Establish Cluster Link from old → new.
2. Wait until lag = 0.
3. Pause producers on old; wait for drain.
4. Cut producers to new.
5. Cut consumers to new (same offsets — no translation).
6. Decommission old.

### Interview Follow-ups

- *"What's the difference between MM2 and Confluent Replicator?"* — Replicator predates MM2, similar idea, Confluent-licensed. MM2 has feature parity for most use cases.
- *"Does replication preserve transactions?"* — Aborted records are filtered if you set `read.isolation.level=read_committed` on the source side. Transactional state itself doesn't cross cluster boundaries.
- *"Why not use Kafka's built-in replication for DR?"* — Native replication requires synchronous ISR — too slow across regions. Cross-cluster replication is async and decoupled.
