# Q: How do Kafka Transactions work?

**Answer:**

Kafka transactions enable **atomic writes across multiple partitions and topics**. They are the foundation for exactly-once semantics (EOS) in the "consume-transform-produce" pattern.

### The Problem
Imagine a stream processing pipeline that reads from topic A, transforms the data, and writes to topic B while also committing consumer offsets. Without transactions, a crash mid-pipeline could result in:
- Data written to topic B but offset not committed → **duplicates on retry**
- Offset committed but data not written to topic B → **data loss**

### How Transactions Work

```java
// 1. Configure transactional producer
Properties props = new Properties();
props.put("transactional.id", "order-processor-1");
props.put("enable.idempotence", "true");

KafkaProducer<String, String> producer = new KafkaProducer<>(props);

// 2. Initialize transactions (called once)
producer.initTransactions();

try {
    // 3. Begin transaction
    producer.beginTransaction();

    // 4. Consume from input topic
    ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(100));

    for (ConsumerRecord<String, String> record : records) {
        // 5. Process and produce to output topic
        String result = transform(record.value());
        producer.send(new ProducerRecord<>("output-topic", record.key(), result));
    }

    // 6. Commit consumer offsets AS PART OF the transaction
    producer.sendOffsetsToTransaction(
        getOffsetsToCommit(records),
        consumer.groupMetadata()
    );

    // 7. Commit transaction (atomic: either ALL writes + offset commit succeed, or NONE)
    producer.commitTransaction();

} catch (Exception e) {
    // 8. Abort transaction (all writes are rolled back)
    producer.abortTransaction();
}
```

### What Happens Atomically

When `commitTransaction()` succeeds, ALL of the following are committed together:
1. All `send()` messages to output topics.
2. The consumer offset commit.

If anything fails, `abortTransaction()` rolls back everything — the output messages are marked as "aborted" and the consumer offsets are not updated.

### Consumer Side: `read_committed`

For consumers to properly participate in transactions:
```properties
isolation.level=read_committed
```

| Isolation Level | Behavior |
|---|---|
| `read_uncommitted` (default) | Consumer sees ALL messages, including those from aborted transactions |
| `read_committed` | Consumer only sees messages from **committed** transactions |

### The `transactional.id`
- Must be **unique per producer instance** but **stable across restarts**.
- When a producer with the same `transactional.id` restarts, Kafka "fences" the old producer — any pending transactions from the old instance are aborted.
- This prevents "zombie" producers from causing duplicates.

> [!TIP]
> Kafka Streams uses transactions internally to provide exactly-once semantics out of the box. You just set `processing.guarantee=exactly_once_v2` and the framework handles all the transactional plumbing automatically.
