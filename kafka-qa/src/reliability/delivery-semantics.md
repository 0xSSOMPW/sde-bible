# Q: What are the different delivery semantics in Kafka?

**Answer:**

This is one of the most important Kafka interview questions. There are three delivery guarantees, and understanding the trade-offs is essential.

### 1. At-Most-Once
Messages may be **lost** but are **never reprocessed**. The consumer commits the offset *before* processing the message.

```
1. Fetch message at offset 42
2. Commit offset 43 ✅
3. Process message... CRASH 💥
4. Restart → reads from offset 43 → message 42 is LOST
```

**When to use:** Metric collection, logging — where occasional loss is acceptable and speed matters most.

### 2. At-Least-Once
Messages are **never lost** but may be **duplicated**. The consumer commits the offset *after* processing the message.

```
1. Fetch message at offset 42
2. Process message ✅
3. Commit offset 43... CRASH 💥 (commit failed)
4. Restart → reads from offset 42 → message 42 is PROCESSED AGAIN
```

**When to use:** Most production systems. Design consumers to be **idempotent** (safe to process twice).

**Idempotent consumer pattern:**
```java
void processOrder(OrderEvent event) {
    // Check if already processed using a deduplication store
    if (processedIds.contains(event.getId())) {
        return; // Skip duplicate
    }
    executeBusinessLogic(event);
    processedIds.add(event.getId());
}
```

### 3. Exactly-Once Semantics (EOS)
Messages are processed **exactly once** — no loss, no duplicates. This is the hardest to achieve and requires specific Kafka features.

**How Kafka achieves EOS:**
1. **Idempotent Producer** (`enable.idempotence=true`) — prevents duplicate writes.
2. **Transactions** (`transactional.id`) — atomic writes across multiple partitions.
3. **Consumer `read_committed` isolation** — consumers only see committed transactional messages.

```properties
# Producer
enable.idempotence=true
transactional.id=my-transaction-id

# Consumer
isolation.level=read_committed
```

**When to use:** Financial systems, inventory management, or when consuming from one topic, processing, and producing to another topic atomically (the "consume-transform-produce" pattern).

### Summary

| Semantic | Data Loss? | Duplicates? | Complexity | Use Case |
|---|---|---|---|---|
| At-most-once | ✅ Possible | ❌ No | Low | Metrics, logs |
| At-least-once | ❌ No | ✅ Possible | Medium | Most production systems |
| Exactly-once | ❌ No | ❌ No | High | Financial, critical data |

> [!IMPORTANT]
> Exactly-once in Kafka is scoped to the **Kafka ecosystem** (producer → broker → consumer within Kafka). It does NOT guarantee exactly-once when writing to external systems (like a database). For end-to-end exactly-once with external systems, you need idempotent consumers or two-phase commit patterns.
