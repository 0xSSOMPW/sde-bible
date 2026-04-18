# Q: How do Consumer Groups and Offsets work in Kafka?

**Answer:**

Consumer groups are Kafka's mechanism for **parallel consumption** and **load balancing**.

### Consumer Groups
A **consumer group** is a set of consumers that cooperate to consume messages from a topic. Each partition is assigned to **exactly one consumer** within a group. This ensures each message is processed once per group.

```
Topic "orders" (4 partitions)
Consumer Group "payment-service" (3 consumers):

  Consumer A ← Partition 0, Partition 1
  Consumer B ← Partition 2
  Consumer C ← Partition 3
```

**Key Rule:** If the number of consumers exceeds the number of partitions, the extra consumers sit idle.

```
4 partitions, 6 consumers:
  Consumer A ← Partition 0
  Consumer B ← Partition 1
  Consumer C ← Partition 2
  Consumer D ← Partition 3
  Consumer E ← IDLE ❌
  Consumer F ← IDLE ❌
```

### Multiple Consumer Groups
Different consumer groups consume the same topic independently. Each group maintains its own offset position — they don't interfere with each other.

```
Topic "orders" (3 partitions) →
  Consumer Group "payment-service"  → reads all 3 partitions independently
  Consumer Group "email-service"    → reads all 3 partitions independently
  Consumer Group "analytics"        → reads all 3 partitions independently
```

This is what makes Kafka a **publish-subscribe** system, not just a queue.

### Offsets
An **offset** is a sequential integer that uniquely identifies each message within a partition. Offsets are how consumers track what they've already read.

```
Partition 0: [0] [1] [2] [3] [4] [5] [6] [7] [8]
                                  ↑
                          Consumer's current offset = 5
                          (has read 0-4, will read 5 next)
```

### Where Are Offsets Stored?
Consumer offsets are committed to an internal Kafka topic called **`__consumer_offsets`** (50 partitions by default). This is a regular compacted topic managed by Kafka itself.

```bash
# Check committed offsets for a group
kafka-consumer-groups.sh --bootstrap-server localhost:9092 \
    --describe --group payment-service
```

Output:
```
GROUP             TOPIC    PARTITION  CURRENT-OFFSET  LOG-END-OFFSET  LAG
payment-service   orders   0          1024            1030            6
payment-service   orders   1          987             987             0
```

> [!IMPORTANT]
> The `LAG` column is critical for monitoring. It shows how many unprocessed messages are waiting. Increasing lag = consumer can't keep up = potential backpressure issue.
