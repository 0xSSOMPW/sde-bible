# Q: Queue vs Pub/Sub vs Log — which messaging model when?

**Answer:**

"Messaging" is three distinct patterns with very different semantics:

- **Queue**: one producer, one consumer per message. Work distribution.
- **Pub/Sub**: one producer, many consumers per message. Fan-out.
- **Log**: ordered, replayable record. Pub/sub + history.

Get the model right; the broker choice (Rabbit, SQS, Kafka) follows.

### Queue

```
producer ──► [ msg1, msg2, msg3 ] ──► consumer1 (gets msg1)
                                    ──► consumer2 (gets msg2)
                                    ──► consumer3 (gets msg3)
```

Each message delivered to **exactly one consumer**. Workers compete; throughput scales with workers.

Used for: background jobs, email sending, image processing, async workflows.

Brokers: RabbitMQ, AWS SQS, Google Pub/Sub (pull mode), Beanstalkd, NATS JetStream.

Semantics:
- **At-least-once** is the typical default; consumer ACKs each message after processing.
- **Visibility timeout**: message hidden after delivery; redelivered if not ACKed in window.
- **DLQ**: after N failures, route to dead-letter queue for inspection.

### Pub/Sub

```
producer ──► topic ──► subscriber A (sees all messages)
                   ──► subscriber B (sees all messages)
                   ──► subscriber C (sees all messages)
```

Each message delivered to **every subscriber**. Subscribers usually independent — A processing doesn't affect B.

Used for: event notifications, cache invalidation, real-time updates.

Brokers: Redis Pub/Sub (ephemeral), Google Pub/Sub (push), MQTT (IoT), Kafka (topics with consumer groups).

Semantics:
- Most pub/sub is at-least-once.
- If a subscriber is offline, may miss messages (depending on broker).
- Filtering: SNS message filters, AMQP routing keys.

### Log

```
producer ──► append-only log ──► consumer A reads from offset 0
                              ──► consumer B reads from offset 0
                              ──► consumer C reads from offset 1000

   [m1][m2][m3][m4][m5][m6][m7]...   ← persistent, ordered
```

A durable, ordered record. Consumers maintain their own offset; can replay from any point. Multiple independent consumer groups read the same log.

Used for: event sourcing, stream processing, audit trails, CDC, data pipelines.

Brokers: Apache Kafka, AWS Kinesis, Apache Pulsar, Redpanda.

Semantics:
- Ordering preserved **per partition**.
- Retention by time or size, not by ACK.
- Replay: a new consumer can start from offset 0 and reprocess history.

### The Three Compared

| Aspect | Queue | Pub/Sub | Log |
|--------|-------|---------|-----|
| Delivery | One per consumer | One per subscriber | One per consumer group |
| Retention | Until ACKed | Volatile or short | Days/weeks (configurable) |
| Replay | No | No | Yes (by offset) |
| Order | Loose (queue order, but workers parallel) | Per-subscriber | Per partition |
| Throughput | Medium | Medium | Very high |
| Best for | Work distribution | Fan-out notifications | Event streams, audit |

### Picking the Right One

Decision tree:

```
Need persistent history? ─── yes ──► Log (Kafka)
              │
              no
              │
Same message to many independent systems? ─── yes ──► Pub/Sub
              │
              no
              │
Multiple workers competing? ─── yes ──► Queue
```

Combinations are common:
- Producer publishes to Kafka (log) → many consumer groups read.
- Each consumer group might internally distribute work via SQS (queue) for fan-out within the service.

### Delivery Guarantees

| Guarantee | Meaning | Cost |
|-----------|---------|------|
| At-most-once | Send and forget; may lose | Fast |
| At-least-once | Retry until ACK; may duplicate | Most common default |
| Exactly-once | One effect, even with retries | Requires idempotency + transactions |

Most brokers do **at-least-once**. Exactly-once is achieved via consumer-side idempotency keys.

See [Delivery Semantics](./delivery-semantics.md).

### Ordering

Strict global ordering = single producer + single consumer. Doesn't scale.

Practical: ordering **per key**.
- Kafka: same key → same partition → ordered.
- SQS FIFO: messages share a `MessageGroupId` → ordered within group.
- Pulsar: keyed_shared subscriptions.

Cross-key ordering: not guaranteed; design your business logic to be commutative or sequenced by app.

### Push vs Pull

| Push | Pull |
|------|------|
| Broker pushes to consumer | Consumer pulls from broker |
| Broker tracks delivery | Consumer tracks offset |
| Easy for slow consumers? Hard (backpressure issues) | Easy (consumer paces) |
| Examples: SNS, webhooks | Kafka, SQS (long-poll) |

Pull-based usually handles backpressure better — consumer reads at its own pace.

### Backpressure

What happens when the consumer can't keep up?

| Broker | Default behavior |
|--------|------------------|
| Kafka | Lag grows; producer not impacted (unless retention deletes unread data) |
| SQS | Queue grows up to limits; visibility timeouts trigger retries |
| RabbitMQ | Producers eventually throttled (flow control); messages discarded if no DLQ |
| Redis Pub/Sub | Messages dropped for slow subscribers (no buffering) |

See [Backpressure](./backpressure.md).

### When Not to Use a Broker

- Synchronous request/response with low fanout — just call the service.
- Workflows requiring strict ordering across many entities — durable workflow engine (Temporal, Camunda).
- Sub-millisecond replies — broker hops add latency.

### Common Mistakes

| Mistake | Reality |
|---------|---------|
| Treating Kafka as a queue (one consumer group) | Works but wastes its strengths; SQS is cheaper |
| Treating SQS as a log | Can't replay; design events around at-least-once + idempotency |
| Redis Pub/Sub for critical events | Drops messages for slow subscribers; no durability |
| Using a queue for fan-out (one queue per subscriber) | Now you have N queues to maintain; use pub/sub |
| Ignoring DLQ | Bad messages cause infinite redelivery loops |

> [!NOTE]
> The hardest model to get right is "exactly-once event processing." Stop fighting it: use at-least-once + idempotent consumers + Idempotency-Key. That's how real systems do it.

### Interview Follow-ups

- *"How do you ensure ordering across services?"* — Per-key ordering via partition/group; for cross-key, sequence via a single owner (saga orchestrator) or accept eventual.
- *"What's the difference between Kafka topic and SQS queue?"* — Topic = log with partitions, consumer groups, retention. Queue = transient buffer, one delivery per message, no replay.
- *"How would you do exactly-once delivery?"* — You don't. Use at-least-once + dedup by `event_id` on the consumer side (DB unique constraint or seen-set).
