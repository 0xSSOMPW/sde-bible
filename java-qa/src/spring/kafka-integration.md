# Q: Kafka with Spring Boot — semantics, ordering, error handling.

**Answer:**

Spring Kafka wraps the official Kafka client with template/listener abstractions. The wrapper is thin; almost every production bug is rooted in misunderstanding *Kafka semantics* (acks, idempotence, transactions, rebalances), not the Spring layer.

### Producer Side

```yaml
spring:
  kafka:
    bootstrap-servers: kafka:9092
    producer:
      acks: all
      retries: 2147483647
      properties:
        enable.idempotence: true
        max.in.flight.requests.per.connection: 5
        delivery.timeout.ms: 120000
        linger.ms: 5
        compression.type: lz4
        # Transactions (optional):
        transactional.id: ${HOSTNAME}-producer
```

Send:

```java
@Component
class Publisher {
    private final KafkaTemplate<String, Order> tmpl;

    public void publish(Order o) {
        tmpl.send("orders", o.id(), o)
            .whenComplete((res, ex) -> {
                if (ex != null) log.error("send failed", ex);
            });
    }
}
```

Key choice matters: same key → same partition → ordering preserved per key. Null key → load-balanced (sticky batching).

### Exactly-Once Producer (Idempotent + Transactions)

`enable.idempotence=true` deduplicates retries within a producer session. `transactional.id` makes writes atomic across topics and the offset commit:

```java
@Transactional("kafkaTransactionManager")
public void process(Order o) {
    template.send("audit", o);
    template.send("invoices", new Invoice(o));
    // both written atomically with the consumer's offset commit
}
```

For exactly-once *end to end*, the consumer must set `isolation.level=read_committed` to skip aborted records.

### Consumer Side

```yaml
spring:
  kafka:
    consumer:
      group-id: orders-app
      enable-auto-commit: false        # always false in production
      auto-offset-reset: earliest
      isolation-level: read_committed
      max-poll-records: 500
    listener:
      type: SINGLE                     # or BATCH
      ack-mode: MANUAL_IMMEDIATE       # or RECORD / BATCH
      concurrency: 3                   # parallel consumers within app
```

Listener:

```java
@KafkaListener(topics = "orders", containerFactory = "orderListenerFactory")
public void onMessage(ConsumerRecord<String, Order> rec, Acknowledgment ack) {
    try {
        process(rec.value());
        ack.acknowledge();
    } catch (RetryableException e) {
        // Spring's container will retry per ErrorHandler config
        throw e;
    } catch (NonRetryableException e) {
        // send to DLT and ack
        dlt.send(rec);
        ack.acknowledge();
    }
}
```

### Ordering and Concurrency

A consumer group reads each partition in order. Across partitions, no order. Inside a consumer instance, Spring's listener container can run `concurrency = N` threads, each consuming one or more partitions.

If you need strict per-key ordering: ensure producer keys by the entity ID, and `concurrency` doesn't exceed partition count (extras are idle). Don't `@Async` or thread-pool inside the listener — that breaks per-partition order.

### Manual Offset Management

```
ack-mode options:
RECORD              # ack after each message
BATCH               # ack after the whole poll batch
MANUAL              # ack when you call ack.acknowledge() — committed on next poll
MANUAL_IMMEDIATE    # ack and commit synchronously NOW (slower but safest)
TIME / COUNT        # ack every N ms or N records
```

Default `BATCH` is fast but risks re-processing the whole batch on crash. `MANUAL_IMMEDIATE` per-record is slow but bounds re-processing to the in-flight record.

### Retry + DLT (Dead Letter Topic)

Spring Kafka has built-in non-blocking retry:

```java
@RetryableTopic(
    attempts = "4",
    backoff = @Backoff(delay = 5000, multiplier = 4.0),
    autoCreateTopics = "true",
    dltStrategy = DltStrategy.FAIL_ON_ERROR,
    include = { TransientException.class }
)
@KafkaListener(topics = "orders")
public void consume(Order o) { ... }

@DltHandler
public void dlt(Order o,
                @Header(KafkaHeaders.EXCEPTION_MESSAGE) String err,
                @Header(KafkaHeaders.ORIGINAL_TOPIC) String orig) {
    alertOps(o, err, orig);
}
```

Creates `orders-retry-0`, `orders-retry-1`, ..., `orders-dlt`. Failed records are routed to retry topics that delay-and-replay rather than blocking the main topic.

### Backpressure

Kafka has no broker-side backpressure. Your consumer must:
- Limit `max.poll.records` to what one poll cycle can process.
- Process synchronously (don't dump to an unbounded `ExecutorService`).
- Monitor `records-lag-max` and alert when it grows.

If your processing is genuinely slow:
- Add partitions to allow more parallel consumers.
- Or shift the slow work to a downstream worker pool with its own queue/topic.

### Rebalance Pitfalls

When a consumer joins/leaves, partitions reassign. By default with `cooperative-sticky`, only affected partitions revoke — but **uncommitted offsets are lost across rebalance**. Always commit before yielding partitions:

```java
@KafkaListener(topics = "orders")
public void onMessage(@Payload Order o,
                      Acknowledgment ack,
                      ConsumerRebalanceListener... ) {
    // process and ack
}
```

Spring exposes a `RebalanceListener` you can implement to flush state on partition revoke.

### Common Mistakes

| Mistake | Reality |
|---------|---------|
| `enable.auto.commit: true` | Offsets advance even on failure — silent data loss |
| `acks=1` for critical writes | One-broker durability — leader loss = data loss |
| `concurrency > partitions` | Extras sit idle |
| Async processing inside `@KafkaListener` | Breaks ordering + ack semantics |
| Catching all exceptions and acking | Failures become silent — design a DLT path |
| Sending to Kafka inside a `@Transactional` JPA method | Two unrelated transaction managers — use `ChainedKafkaTransactionManager` or outbox pattern |

### Transactional Outbox Pattern

Mixing DB commits and Kafka sends safely requires either:

1. **Chained transaction manager** (DB + Kafka, two-phase commit-ish).
2. **Transactional outbox**: write a row to `outbox` table in the same DB transaction, separate poller (Debezium CDC or app-side polling) republishes to Kafka.

Outbox is the recommended production pattern — it survives crashes between DB commit and Kafka send.

### Observability

Micrometer-instrumented out of the box. Watch:

```
spring.kafka.listener.* timer
kafka.consumer.records-lag-max
kafka.consumer.records-consumed-total
kafka.producer.record-send-total
kafka.producer.record-error-total
```

Trace context: enable Spring Cloud Sleuth / Micrometer Tracing and Kafka client interceptors to propagate trace IDs in headers.

> [!NOTE]
> Idiomatic Spring Kafka is thin. Almost every gotcha is Kafka behavior, not Spring sugar. Read the consumer config doc once, top to bottom.

### Interview Follow-ups

- *"How is Kafka's transactional producer different from a DB transaction?"* — Atomic across topic partitions + offset commits. Doesn't span outside Kafka.
- *"Why is `max.in.flight=5` safe with idempotence?"* — Producer sequence numbers let brokers reorder retries correctly. Without idempotence, in-flight > 1 + retries can reorder messages.
- *"What is `ChainedKafkaTransactionManager`?"* — Spring's coordinator that opens DB + Kafka transactions and synchronizes commit/rollback. Not a true XA — best-effort with known failure modes.
