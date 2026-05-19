# Summary

- [Introduction](./README.md)

---

# Architecture

- [Core Concepts]()
  - [What is Kafka & Why Use It?](./architecture/what-is-kafka.md)
  - [Brokers, Topics & Partitions](./architecture/brokers-topics-partitions.md)
  - [ZooKeeper vs KRaft](./architecture/zookeeper-vs-kraft.md)
  - [Replication & ISR](./architecture/replication-isr.md)
  - [Tiered Storage (KIP-405)](./architecture/tiered-storage.md)
  - [Log Segments, Indexes & Page Cache](./architecture/log-segments.md)
  - [KRaft Mode (Without ZooKeeper)](./architecture/kraft-mode.md)

---

# Producers

- [Producer Internals]()
  - [Producer Acks & Durability](./producers/acks-durability.md)
  - [Idempotent Producers](./producers/idempotent-producers.md)
  - [Partitioning Strategies](./producers/partitioning-strategies.md)
  - [Batching & Compression](./producers/batching-compression.md)

---

# Consumers

- [Consumer Internals]()
  - [Consumer Groups & Offsets](./consumers/consumer-groups-offsets.md)
  - [Offset Commit Strategies](./consumers/offset-commit-strategies.md)
  - [Consumer Rebalancing](./consumers/rebalancing.md)
  - [Cooperative Rebalancing & KIP-848](./consumers/cooperative-rebalancing.md)
  - [Consumer Tuning (Throughput, Latency, Safety)](./consumers/consumer-tuning.md)

---

# Reliability & Semantics

- [Delivery Guarantees]()
  - [At-Most-Once, At-Least-Once, Exactly-Once](./reliability/delivery-semantics.md)
  - [Transactions in Kafka](./reliability/transactions.md)
  - [Dead Letter Queues & Retry Topics](./reliability/dead-letter-queue.md)

---

# Performance

- [Performance Tuning]()
  - [Why is Kafka So Fast?](./performance/why-kafka-fast.md)
  - [Consumer Lag & Backpressure](./performance/consumer-lag.md)
  - [Hot Partitions: Detection & Remediation](./performance/hot-partitions.md)

---

# Ecosystem & Operations

- [Kafka Ecosystem]()
  - [Kafka Connect](./ecosystem/kafka-connect.md)
  - [Kafka Streams vs Flink vs Spark](./ecosystem/streams-vs-flink.md)
  - [Schema Registry & Avro](./ecosystem/schema-registry.md)
  - [Schema Evolution & Compatibility](./ecosystem/schema-evolution.md)
  - [Kafka Streams Patterns (KStream, KTable, Joins)](./ecosystem/kafka-streams-patterns.md)
- [Operations]()
  - [Retention & Log Compaction](./operations/retention-compaction.md)
  - [Monitoring & Key Metrics](./operations/monitoring.md)
  - [Kafka vs RabbitMQ vs SQS](./operations/kafka-vs-rabbitmq.md)
  - [Cluster Sizing & Capacity Planning](./operations/cluster-sizing.md)
  - [Securing a Kafka Cluster (TLS, SASL, ACLs)](./operations/security.md)
  - [Multi-Datacenter Replication (MM2 vs Cluster Linking)](./operations/multi-datacenter.md)
