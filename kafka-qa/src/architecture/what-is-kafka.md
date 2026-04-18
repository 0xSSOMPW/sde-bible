# Q: What is Apache Kafka and why would you use it?

**Answer:**

Apache Kafka is an **open-source distributed event streaming platform** originally developed at LinkedIn and donated to the Apache Software Foundation. It is designed for high-throughput, fault-tolerant, real-time data streaming.

### What Kafka Does
At its core, Kafka is a **distributed commit log**. Producers write messages (events) to topics, and consumers read those messages. Unlike traditional message queues, messages in Kafka are **persisted to disk** and can be replayed.

### Why Use Kafka?

**1. Decoupling of Systems**
Instead of service A calling service B directly (tight coupling), A publishes an event to Kafka. Any service interested in that event subscribes independently.

```
Without Kafka:  OrderService → PaymentService → EmailService → AnalyticsService
                (chain of synchronous calls, one failure breaks everything)

With Kafka:     OrderService → [Kafka Topic: "orders"]
                                    ├── PaymentService (consumer)
                                    ├── EmailService (consumer)
                                    └── AnalyticsService (consumer)
```

**2. Extreme Throughput**
Kafka handles **millions of messages per second** with latency under 10ms. LinkedIn processes over 7 trillion messages/day through Kafka.

**3. Durability & Replayability**
Messages are written to disk and replicated across brokers. Consumers can re-read old messages (e.g., replay the last 7 days of events to rebuild a search index).

**4. Ordering Guarantees**
Messages within a single partition are strictly ordered — essential for event sourcing and log-based architectures.

### Common Use Cases
- **Event-driven microservices** — publish domain events between services.
- **Real-time analytics** — stream clickstream data to dashboards.
- **Log aggregation** — centralize logs from thousands of servers.
- **Change Data Capture (CDC)** — stream database changes (via Debezium + Kafka Connect).
- **ETL pipelines** — replace batch processing with real-time streaming.

> [!NOTE]
> Kafka is NOT a traditional message queue (like RabbitMQ or SQS). It's a **distributed log** that happens to be excellent at messaging. The key difference: messages aren't deleted after consumption — they're retained based on a time or size policy.
