# Q: What is Consumer Lag and how do you handle backpressure?

**Answer:**

**Consumer lag** is the difference between the latest message offset in a partition (log-end offset) and the consumer's current committed offset. It tells you how far behind a consumer is.

### Measuring Lag

```bash
kafka-consumer-groups.sh --bootstrap-server localhost:9092 \
    --describe --group my-service

GROUP       TOPIC    PARTITION  CURRENT-OFFSET  LOG-END-OFFSET  LAG
my-service  orders   0          4500            5000            500
my-service  orders   1          3200            3200            0
my-service  orders   2          2800            4100            1300
```

**Healthy:** LAG = 0 or near-zero and stable.
**Unhealthy:** LAG is increasing over time = consumer can't keep up with producers.

### Causes of Growing Lag

1. **Slow processing** — each message takes too long (external API calls, heavy computation).
2. **Insufficient consumers** — fewer consumers than partitions.
3. **Frequent rebalances** — consumers pausing during rebalancing.
4. **GC pauses** — long garbage collection stops in JVM-based consumers.
5. **Skewed partitions** — one partition has significantly more data due to hot keys.

### Strategies to Handle Backpressure

**1. Scale consumers horizontally**
Add more consumer instances (up to the number of partitions):
```
Before: 2 consumers for 6 partitions (3 partitions each)
After:  6 consumers for 6 partitions (1 partition each)
```

**2. Increase partitions**
More partitions = more parallelism. But be cautious of the repartitioning trap (breaks key ordering for existing data).

**3. Optimize processing**
- Process messages asynchronously (decouple consumption from processing).
- Use batch processing instead of one-at-a-time.
- Cache external service responses.

**4. Tune consumer configs**
```properties
max.poll.records=100        # Fewer records per poll = less time per batch
fetch.min.bytes=1           # Don't wait for large fetches
max.poll.interval.ms=600000 # More time allowed between polls
```

**5. Dead Letter Queue (DLQ)**
If a specific message consistently fails processing, send it to a DLQ topic instead of blocking the consumer:
```java
try {
    processMessage(record);
} catch (Exception e) {
    producer.send(new ProducerRecord<>("orders.dlq", record.key(), record.value()));
    // Continue processing next message
}
```

### Monitoring Lag

Critical metrics to alert on:
- **`kafka.consumer.lag`** — absolute lag (messages behind).
- **`kafka.consumer.lag_rate`** — rate of lag change (is it growing?).
- **Consumer group state** — `STABLE`, `REBALANCING`, `DEAD`.

Tools: Burrow (LinkedIn), Kafka Lag Exporter (Prometheus), or built-in `kafka-consumer-groups.sh`.

> [!CAUTION]
> A sudden spike in consumer lag often precedes a production incident. Set up alerts for when lag exceeds a threshold (e.g., >10,000 messages) OR when lag is consistently increasing over a 5-minute window.
