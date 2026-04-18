# Summary

- [Introduction](./README.md)

---

# Architecture

- [Core Concepts]()
  - [What is Kafka & Why Use It?](./architecture/what-is-kafka.md)
  - [Brokers, Topics & Partitions](./architecture/brokers-topics-partitions.md)
  - [ZooKeeper vs KRaft](./architecture/zookeeper-vs-kraft.md)
  - [Replication & ISR](./architecture/replication-isr.md)

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

---

# Reliability & Semantics

- [Delivery Guarantees]()
  - [At-Most-Once, At-Least-Once, Exactly-Once](./reliability/delivery-semantics.md)
  - [Transactions in Kafka](./reliability/transactions.md)

---

# Performance

- [Performance Tuning]()
  - [Why is Kafka So Fast?](./performance/why-kafka-fast.md)
  - [Consumer Lag & Backpressure](./performance/consumer-lag.md)

---

# Ecosystem & Operations

- [Kafka Ecosystem]()
  - [Kafka Connect](./ecosystem/kafka-connect.md)
  - [Kafka Streams vs Flink vs Spark](./ecosystem/streams-vs-flink.md)
  - [Schema Registry & Avro](./ecosystem/schema-registry.md)
- [Operations]()
  - [Retention & Log Compaction](./operations/retention-compaction.md)
  - [Monitoring & Key Metrics](./operations/monitoring.md)
  - [Kafka vs RabbitMQ vs SQS](./operations/kafka-vs-rabbitmq.md)
