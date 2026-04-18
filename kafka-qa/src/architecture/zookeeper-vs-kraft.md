# Q: What is the difference between ZooKeeper and KRaft mode?

**Answer:**

This is a hot interview topic because Kafka is in the middle of a major architectural transition.

### ZooKeeper Mode (Legacy)
Historically, Kafka relied on **Apache ZooKeeper** — a separate distributed coordination service — to manage cluster metadata:
- **Broker registration** (which brokers are alive)
- **Controller election** (one broker is the "controller" that manages partition leadership)
- **Topic configuration** (partition count, replication factor, ACLs)
- **Consumer group offsets** (in older versions; now stored in Kafka itself)

```
┌─────────────────────┐
│  ZooKeeper Ensemble  │  (3-5 separate servers)
│  ┌───┐ ┌───┐ ┌───┐  │
│  │ZK1│ │ZK2│ │ZK3│  │
│  └───┘ └───┘ └───┘  │
└─────────┬───────────┘
          │ metadata
┌─────────▼───────────┐
│    Kafka Cluster     │
│  ┌──┐ ┌──┐ ┌──┐     │
│  │B0│ │B1│ │B2│     │
│  └──┘ └──┘ └──┘     │
└─────────────────────┘
```

### KRaft Mode (New, ZooKeeper-Free)
Starting with Kafka 3.3 (production-ready in 3.5+), Kafka can run **without ZooKeeper** using an internal Raft-based consensus protocol called **KRaft** (Kafka Raft).

In KRaft mode, a subset of Kafka brokers act as **controllers** and manage all metadata internally using a replicated metadata log (`__cluster_metadata` topic).

```
┌─────────────────────────────┐
│        Kafka Cluster        │
│  ┌────────┐  ┌──┐  ┌──┐    │
│  │B0 (ctrl)│  │B1│  │B2│   │
│  │B1 (ctrl)│  └──┘  └──┘   │
│  │B2 (ctrl)│                │
│  └────────┘                 │
│  (controllers embedded)     │
└─────────────────────────────┘
```

### Why the Migration?

| Concern | ZooKeeper | KRaft |
|---|---|---|
| **Operational complexity** | Separate cluster to deploy, monitor, upgrade | All-in-one, no external dependency |
| **Partition limit** | ~200K partitions (ZK bottleneck) | Millions of partitions |
| **Controller failover** | 10-30 seconds (ZK session timeout) | Seconds (Raft leader election) |
| **Metadata propagation** | Asynchronous, eventual consistency | Replicated log, strongly consistent |
| **Security** | Separate ACL system | Unified with Kafka's security |

### Current Status
- **ZooKeeper mode**: Deprecated as of Kafka 3.5. Will be removed entirely in Kafka 4.0.
- **KRaft mode**: Production-ready. All new deployments should use KRaft.

> [!TIP]
> In interviews, mentioning the KRaft migration shows you're up-to-date with the Kafka ecosystem. If asked "how does Kafka manage metadata?", mention *both* modes and note that ZooKeeper is being phased out.
