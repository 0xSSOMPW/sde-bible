# Q: What is Kafka Tiered Storage (KIP-405) and when should you use it?

**Answer:**

Tiered Storage decouples Kafka's **compute** (brokers) from its **storage** (long-term retention) by moving older log segments off broker-local disks into cheaper remote object storage (S3, GCS, Azure Blob, HDFS).

### The Problem It Solves

Pre-KIP-405, every byte you wanted to retain lived on the broker's local disk. That forced uncomfortable tradeoffs:

- Want 30-day retention for replay? Buy 30 days of SSD on every broker.
- Replication factor 3 multiplies that storage cost by 3x.
- Adding a broker triggers expensive partition reassignment to rebalance terabytes of cold data.
- A broker failure means hours of re-replication for data nobody is actively reading.

### The Architecture

```
        ┌─────────────────────────────────────────┐
        │              Kafka Broker               │
        │                                         │
        │   ┌─────────────┐   ┌──────────────┐    │
        │   │ Local Tier  │   │ Remote Log   │    │
        │   │ (hot data)  │   │ Manager      │    │
        │   │  SSD/NVMe   │   │ (RLM)        │    │
        │   └─────────────┘   └──────┬───────┘    │
        └─────────────────────────────┼───────────┘
                                      │
                              ┌───────▼────────┐
                              │ Remote Tier    │
                              │ S3 / GCS / ... │
                              │ (cold data)    │
                              └────────────────┘
```

- **Local tier**: Active segment + recent closed segments. Serves the tail of the log — what producers write and consumers usually read.
- **Remote tier**: Closed segments older than `local.retention.ms` are uploaded by the Remote Log Manager. The local copy is then deleted.
- **Reads** for offsets in remote segments are fetched transparently — the consumer doesn't know or care where bytes live.

### Configuration

Cluster level:

```properties
remote.log.storage.system.enable=true
remote.log.storage.manager.class.name=org.apache.kafka.server.log.remote.storage.RemoteLogManager
remote.log.metadata.manager.class.name=org.apache.kafka.server.log.remote.metadata.storage.TopicBasedRemoteLogMetadataManager
```

Topic level:

```bash
kafka-configs.sh --alter --topic events \
  --add-config remote.storage.enable=true,\
local.retention.ms=86400000,\
retention.ms=2592000000
```

- `retention.ms`: total retention (local + remote). Here, 30 days.
- `local.retention.ms`: how much stays on broker disk. Here, 1 day.

### When to Use It

Good fit:
- Long-retention topics (compliance, audit, replay, ML training data).
- Topics where 95% of reads are tail reads and historical reads are rare.
- You want elastic brokers — adding/removing nodes shouldn't shuffle terabytes.

Bad fit:
- Low-retention, high-throughput pipelines (you'd just round-trip to S3 for no reason).
- Workloads with frequent historical scans (S3 GET latency >> NVMe).
- Compacted topics — KIP-405 currently only supports delete-cleanup-policy topics.

### Tradeoffs

| Aspect | Local-only | Tiered |
|--------|-----------|--------|
| Storage cost | High (SSD × RF) | Low (S3 single-copy + lifecycle) |
| Tail-read latency | µs | µs (same) |
| Historical-read latency | µs–ms | 10s–100s of ms |
| Broker scaling | Slow (rebalance TBs) | Fast (only hot data moves) |
| Failure recovery | Re-replicate everything | Re-replicate hot tier only |

> [!NOTE]
> Confluent Cloud and AWS MSK both ship Tiered Storage with proprietary remote tiers. KIP-405 is the open-source baseline — production-ready in Apache Kafka 3.6+.

### Interview Follow-ups

- *"What happens to remote segments if a topic is deleted?"* — RLM issues asynchronous delete of remote objects; orphans are possible during failure, hence lifecycle policies.
- *"How is metadata about remote segments stored?"* — In an internal topic `__remote_log_metadata` (default impl).
- *"Does it affect producer throughput?"* — No; producers only write to local tier. Upload is async.
