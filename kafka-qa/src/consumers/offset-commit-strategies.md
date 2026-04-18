# Q: What are the different Offset Commit Strategies?

**Answer:**

How and when a consumer commits its offset determines what happens when the consumer crashes and restarts. This directly affects delivery semantics.

### Auto-Commit (Default)
Offsets are committed automatically at a fixed interval, regardless of whether messages have been processed.

```properties
enable.auto.commit=true        # Default
auto.commit.interval.ms=5000   # Every 5 seconds
```

**The Problem:**
1. Consumer fetches messages at offset 100-110.
2. Auto-commit fires, committing offset 110.
3. Consumer crashes while processing message 105.
4. Consumer restarts, reads from offset 110 → **messages 105-109 are lost**.

This creates **at-most-once** semantics: messages can be lost, but are never reprocessed.

### Manual Commit (Synchronous)
The application explicitly commits after successfully processing messages.

```java
consumer.poll(Duration.ofMillis(100));
// Process messages...
consumer.commitSync(); // Blocks until commit is confirmed
```

**Trade-off:** If the consumer crashes *after* processing but *before* committing, it will reprocess those messages on restart → **at-least-once** semantics (duplicates possible, but no data loss).

### Manual Commit (Asynchronous)
Same as synchronous but non-blocking:

```java
consumer.commitAsync((offsets, exception) -> {
    if (exception != null) {
        log.error("Commit failed", exception);
    }
});
```

**Trade-off:** Higher throughput, but if the commit fails silently, you may reprocess messages.

### Best Practice: Sync + Async Hybrid

```java
try {
    while (true) {
        ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(100));
        for (ConsumerRecord<String, String> record : records) {
            processRecord(record);
        }
        consumer.commitAsync(); // Fast, non-blocking for normal flow
    }
} catch (Exception e) {
    log.error("Consumer error", e);
} finally {
    consumer.commitSync(); // Guaranteed commit on shutdown
    consumer.close();
}
```

### Commit Granularity

You can also commit offsets for specific partitions:
```java
Map<TopicPartition, OffsetAndMetadata> offsets = new HashMap<>();
offsets.put(new TopicPartition("orders", 0), new OffsetAndMetadata(lastOffset + 1));
consumer.commitSync(offsets);
```

### Summary

| Strategy | Delivery Semantics | Risk |
|---|---|---|
| Auto-commit | At-most-once | Message loss after crash |
| Manual after processing | At-least-once | Duplicate processing after crash |
| Transactional (EOS) | Exactly-once | Highest complexity |

> [!TIP]
> Most production systems use **manual commits with at-least-once semantics** and design their consumers to be **idempotent** (processing the same message twice produces the same result). This is simpler and more reliable than attempting exactly-once.
