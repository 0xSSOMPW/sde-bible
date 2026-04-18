# Q: How does Kafka decide which partition a message goes to?

**Answer:**

The partition assignment strategy determines message ordering and parallelism.

### Partitioning Strategies

**1. Key-Based Partitioning (Default when key is provided)**
When a message has a key, Kafka applies `murmur2(key) % numPartitions` to determine the partition. All messages with the **same key always go to the same partition**, guaranteeing ordering for that key.

```java
producer.send(new ProducerRecord<>("orders", "user-123", orderEvent));
// All events for "user-123" go to the same partition → strict ordering
```

**2. Round-Robin (Default when key is null, Kafka < 2.4)**
Messages without a key are distributed across partitions in a round-robin fashion.

**3. Sticky Partitioning (Default when key is null, Kafka ≥ 2.4)**
Instead of round-robin per message, the producer "sticks" to one partition for the duration of a batch, then switches. This significantly improves batching efficiency and throughput.

```
Round-Robin:  P0, P1, P2, P0, P1, P2  (small batches, many network calls)
Sticky:       P0, P0, P0, P1, P1, P1  (full batches, fewer network calls)
```

**4. Custom Partitioner**
You can implement your own partitioning logic:

```java
public class GeoPartitioner implements Partitioner {
    @Override
    public int partition(String topic, Object key, byte[] keyBytes,
                         Object value, byte[] valueBytes, Cluster cluster) {
        String region = extractRegion(key);
        if ("us-east".equals(region)) return 0;
        if ("eu-west".equals(region)) return 1;
        return 2; // default
    }
}
```

### The Repartitioning Trap

> [!CAUTION]
> If you **add partitions** to an existing topic, the key-to-partition mapping changes (`murmur2(key) % newNumPartitions`). Messages for the same key may suddenly go to a different partition, breaking ordering guarantees for in-flight data. This is why **you should plan partition counts carefully upfront**.

### Choosing the Right Strategy

| Strategy | Key Provided? | Ordering | Use Case |
|---|---|---|---|
| Key-based hash | ✅ Yes | Per-key ordering | User events, order processing |
| Sticky (null key) | ❌ No | None | Logs, metrics, high-throughput |
| Custom | Either | Custom logic | Geo-routing, priority lanes |
