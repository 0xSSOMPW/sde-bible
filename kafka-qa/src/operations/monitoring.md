# Q: What are the key metrics to monitor in a Kafka cluster?

**Answer:**

Monitoring is essential for maintaining a healthy Kafka cluster. Here are the critical metrics organized by component.

### Broker Metrics

| Metric | What It Tells You | Alert Threshold |
|---|---|---|
| **UnderReplicatedPartitions** | Partitions where followers are behind the leader | > 0 for sustained period |
| **ActiveControllerCount** | Number of active controllers in the cluster | Should always be exactly 1 |
| **OfflinePartitionsCount** | Partitions with no leader (completely unavailable) | > 0 = critical |
| **RequestHandlerAvgIdlePercent** | How busy the broker's request handler threads are | < 20% = broker overloaded |
| **NetworkProcessorIdlePercent** | Network thread utilization | < 30% = network bottleneck |
| **LogFlushLatencyMs** | Time to flush logs to disk | Spikes indicate disk issues |

### Producer Metrics

| Metric | What It Tells You | Alert Threshold |
|---|---|---|
| **record-send-rate** | Messages sent per second | Sudden drop = producer issue |
| **record-error-rate** | Failed sends per second | > 0 = investigate |
| **batch-size-avg** | Average batch size | Too small = suboptimal batching |
| **request-latency-avg** | Avg time broker takes to respond | > 100ms = potential issue |

### Consumer Metrics

| Metric | What It Tells You | Alert Threshold |
|---|---|---|
| **records-lag-max** | Maximum lag across all partitions | Consistently increasing |
| **records-consumed-rate** | Messages consumed per second | Sudden drop = consumer issue |
| **commit-latency-avg** | Time to commit offsets | Spikes indicate issues |
| **rebalance-rate** | How often the group rebalances | High rate = configuration issue |

### Monitoring Stack

```
Kafka (JMX Metrics)
    ↓
Prometheus (JMX Exporter)
    ↓
Grafana (Dashboards + Alerts)
```

**Popular Tools:**
- **Prometheus + JMX Exporter**: Industry standard for metric collection.
- **Grafana**: Visualization and alerting.
- **Burrow**: LinkedIn's tool specifically for consumer lag monitoring.
- **Kafka Manager / AKHQ**: Web UI for cluster management.
- **Confluent Control Center**: Commercial monitoring (Confluent Platform).

### Critical Alerts to Set Up

```yaml
# Example Prometheus alerting rules
groups:
  - name: kafka-alerts
    rules:
      - alert: KafkaOfflinePartitions
        expr: kafka_server_replicamanager_offline_partitions_count > 0
        for: 1m
        labels:
          severity: critical

      - alert: KafkaConsumerLagHigh
        expr: kafka_consumer_group_lag > 10000
        for: 5m
        labels:
          severity: warning

      - alert: KafkaUnderReplicatedPartitions
        expr: kafka_server_replicamanager_under_replicated_partitions > 0
        for: 5m
        labels:
          severity: warning
```

> [!TIP]
> In interviews, the most impactful metrics to mention are **UnderReplicatedPartitions** (replication health), **consumer lag** (processing health), and **OfflinePartitionsCount** (availability). These cover the three biggest operational concerns: data durability, throughput, and uptime.
