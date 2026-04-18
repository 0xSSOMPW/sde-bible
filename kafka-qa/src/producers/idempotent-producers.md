# Q: What is an Idempotent Producer in Kafka?

**Answer:**

An **idempotent producer** guarantees that even if a message is sent multiple times (due to retries), it is written to the Kafka log **exactly once** per partition. This eliminates duplicate messages caused by network errors.

### The Problem Without Idempotency
1. Producer sends message A to broker.
2. Broker writes message A and sends an ACK.
3. The ACK is lost due to a network glitch.
4. Producer thinks the write failed, so it **retries** message A.
5. Broker writes message A **again** → **duplicate**.

### How Idempotency Works
When enabled, Kafka assigns each producer a unique **Producer ID (PID)** and each message gets a **sequence number** per partition.

The broker tracks the latest sequence number for each PID+partition pair. If a message arrives with a sequence number that has already been written, the broker **silently discards the duplicate** and returns a success ACK.

```
Producer (PID=42) → Partition 0:
  Msg(seq=0) → Written ✅
  Msg(seq=1) → Written ✅
  Msg(seq=1) → Duplicate, discarded! (but ACK sent) ✅
  Msg(seq=2) → Written ✅
```

### Enabling Idempotency

```properties
# Producer config
enable.idempotence=true

# These are automatically set when idempotence is enabled:
acks=all
retries=Integer.MAX_VALUE
max.in.flight.requests.per.connection=5  # (was 1 in older versions)
```

> [!NOTE]
> Since Kafka 3.0, `enable.idempotence=true` is the **default**. You don't need to explicitly set it in newer versions.

### Scope and Limitations

| Feature | Idempotent Producer | Transactional Producer |
|---|---|---|
| **Dedup scope** | Single partition, single session | Cross-partition, cross-session |
| **Survives restart** | ❌ (new PID on restart) | ✅ (uses `transactional.id`) |
| **Use case** | Prevent network-retry duplicates | Exactly-once across partitions |

Idempotency alone does NOT provide exactly-once semantics across multiple partitions or consumer-producer chains. For that, you need **transactions** (covered in the Reliability section).

> [!TIP]
> In interviews, the key insight is: idempotency prevents duplicates from **retries within a single producer session**. It does NOT prevent duplicates from application-level retries (e.g., your service crashes and replays the same business logic). For that, you need application-level deduplication or Kafka transactions.
