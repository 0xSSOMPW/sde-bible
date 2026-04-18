# Q: When would you choose Kafka over RabbitMQ or SQS?

**Answer:**

This is a common architectural decision question. Each messaging system serves different primary use cases.

### Apache Kafka
A **distributed event streaming platform** designed as a durable commit log.

**Best for:**
- High-throughput event streaming (millions of msg/sec)
- Event sourcing and CQRS architectures
- Log aggregation
- Real-time analytics pipelines
- When consumers need to **replay** old messages
- When multiple independent consumers need the same data

### RabbitMQ
A **traditional message broker** that implements AMQP (Advanced Message Queuing Protocol).

**Best for:**
- Request/reply patterns (RPC over messaging)
- Complex routing logic (topic exchanges, headers, fanout)
- Priority queues (some messages should be processed first)
- When messages should be **deleted after consumption**
- Smaller scale (thousands, not millions, of msg/sec)
- When you need per-message acknowledgement and fine-grained delivery control

### Amazon SQS
A **fully managed message queue** on AWS.

**Best for:**
- Teams that don't want to operate infrastructure
- Simple producer-consumer patterns
- Variable/bursty workloads (auto-scales transparently)
- Dead letter queue support out of the box
- When tight AWS integration is needed (Lambda triggers, IAM)

### Comparison Table

| Feature | Kafka | RabbitMQ | SQS |
|---|---|---|---|
| **Model** | Distributed log | Message broker | Managed queue |
| **Throughput** | Millions/sec | Thousands/sec | Variable (managed) |
| **Message retention** | Days/weeks (configurable) | Until consumed | 4 days (max 14) |
| **Replay** | ✅ Yes | ❌ No | ❌ No |
| **Ordering** | Per-partition | Per-queue | FIFO variant only |
| **Consumer groups** | ✅ Built-in | Manual (competing consumers) | ✅ Built-in |
| **Delivery semantics** | At-least-once, exactly-once | At-least-once, at-most-once | At-least-once |
| **Complex routing** | ❌ Topic-based only | ✅ Exchanges, bindings, headers | ❌ Simple |
| **Priority queues** | ❌ No | ✅ Yes | ❌ No |
| **Operations** | Self-managed or Confluent Cloud | Self-managed or CloudAMQP | Fully managed |
| **Protocol** | Custom binary | AMQP, STOMP, MQTT | HTTP/SQS API |

### Decision Framework

```
Need replay / event sourcing?          → Kafka
High throughput (>100K msg/sec)?       → Kafka
Multiple independent consumers?       → Kafka
Complex routing (headers, priorities)? → RabbitMQ
Request/reply (RPC) pattern?           → RabbitMQ
Don't want to manage infrastructure?  → SQS (or Confluent Cloud/CloudAMQP)
Simple queue, AWS ecosystem?           → SQS
```

> [!NOTE]
> These are not mutually exclusive. Many production architectures use **Kafka for event streaming** (inter-service communication) AND **SQS/RabbitMQ for task queues** (background job processing). Using the right tool for each specific use case is the mark of a mature architecture.
