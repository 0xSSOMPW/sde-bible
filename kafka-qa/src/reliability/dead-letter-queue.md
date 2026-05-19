# Q: How do you implement Dead Letter Queues (DLQ) and retry topics in Kafka?

**Answer:**

Unlike RabbitMQ or SQS, Kafka has **no native DLQ primitive**. A DLQ in Kafka is just *another topic* you publish to when processing fails — but the patterns around routing, retries, and replay are what make or break production systems.

### The Core Problem

A consumer pulls a poison message — bad schema, downstream 500, programmer error. Options:

- **Block forever**: retry in place. Partition stops moving. Lag grows. Other partitions keep going. Eventually pages someone.
- **Skip and commit**: data loss.
- **Park it**: write to a DLQ topic, commit offset, keep moving.

DLQ = "park it, deal with it later."

### Topology

```
              ┌─────────────────────────┐
              │       orders            │
              └──────────┬──────────────┘
                         │
                  ┌──────▼───────┐
                  │  Consumer    │
                  └──┬─────────┬─┘
                     │ ok      │ fail
                     │         │
                ┌────▼───┐  ┌──▼──────────────┐
                │ commit │  │ orders.retry.5s │──┐
                └────────┘  └─────────────────┘  │
                                                 │ still fails
                            ┌─────────────────┐  │
                            │ orders.retry.30s│◀─┘
                            └────────┬────────┘
                                     │ still fails
                            ┌────────▼────────┐
                            │   orders.dlq    │
                            └─────────────────┘
```

Tiered retry topics let you back off without blocking the main topic.

### Pattern 1: Plain DLQ (no retry)

```java
try {
    process(record);
} catch (RetryableException e) {
    // also goes to DLQ in this simple variant
    producer.send(new ProducerRecord<>("orders.dlq", record.key(), record.value()));
} catch (NonRetryableException e) {
    producer.send(new ProducerRecord<>("orders.dlq", record.key(), record.value()));
}
consumer.commitSync();
```

Always attach metadata in headers — *why* it failed, *which* consumer, original offset/partition, attempt count.

```java
record.headers().add("error-class", e.getClass().getName().getBytes());
record.headers().add("error-message", e.getMessage().getBytes());
record.headers().add("original-topic", "orders".getBytes());
record.headers().add("attempt", String.valueOf(attempt).getBytes());
```

### Pattern 2: Non-blocking Tiered Retry (Confluent / Spring style)

Spring Kafka's `@RetryableTopic` or Confluent's parallel-consumer auto-create topics like:

```
orders
orders-retry-0    (delay 5s)
orders-retry-1    (delay 30s)
orders-retry-2    (delay 5m)
orders-dlt        (terminal)
```

Each retry topic has a consumer that sleeps until `record.timestamp + delay`, then re-attempts. Lower partitions don't get blocked by one bad record.

```java
@RetryableTopic(
    attempts = "4",
    backoff = @Backoff(delay = 5000, multiplier = 6.0),
    dltTopicSuffix = "-dlt",
    autoCreateTopics = "true"
)
@KafkaListener(topics = "orders")
public void consume(Order o) { ... }

@DltHandler
public void dlt(Order o, @Header(KafkaHeaders.EXCEPTION_MESSAGE) String err) { ... }
```

### Pattern 3: Kafka Connect Sink DLQ

Built in. Set:

```properties
errors.tolerance=all
errors.deadletterqueue.topic.name=connect.dlq
errors.deadletterqueue.context.headers.enable=true
```

Connect routes failed records with full error context in headers.

### Common Mistakes

| Mistake | Fix |
|--------|-----|
| Same partition count for DLQ as main → hot partitions during incidents | Size DLQ for *spike* throughput, not steady state |
| No alerting on DLQ ingest rate | Alert if `dlq.records.per.sec > 0` for >N min |
| Replay tool dumps DLQ back to main without fixing schema | Replay must be a separate, gated workflow |
| Committing offset *before* DLQ produce succeeds | Order: produce → ack → commit |

### Replay Strategy

Don't auto-replay. A typical replay pipeline:

1. Engineer reads DLQ headers, identifies root cause.
2. Fix consumer code, deploy.
3. Run a *replay job* that reads `orders.dlq` and re-produces matching records to `orders`.
4. Move the replayed records to `orders.dlq.replayed.<date>` for audit.

> [!NOTE]
> A DLQ that nobody monitors is a data leak with extra steps. Treat DLQ depth as a first-class SLI.

### Interview Follow-ups

- *"Why not just `pause()` the partition?"* — Works for transient failures, but lag balloons and the consumer becomes a state machine you have to babysit. DLQ trades latency for liveness.
- *"How to preserve ordering with DLQ?"* — You can't perfectly. Tiered retries reorder by design. If strict ordering matters, block (pause) and page; don't use DLQ.
- *"What about exactly-once?"* — Wrap produce-to-DLQ + commit-offset in a transaction (`isolation.level=read_committed` on downstream).
