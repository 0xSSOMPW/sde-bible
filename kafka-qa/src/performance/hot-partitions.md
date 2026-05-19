# Q: What are hot partitions, how do you detect them, and how do you fix them?

**Answer:**

A **hot partition** is one that receives or serves a disproportionate share of traffic. Because a single partition is consumed by exactly one consumer in a group, a hot partition becomes the parallelism ceiling of your entire pipeline — adding consumers won't help.

### Why Hot Partitions Happen

Default Kafka partitioner: `hash(key) % numPartitions`. Three failure modes:

1. **Skewed key distribution.** One tenant, one device ID, one product ID generates 10x traffic. Hash doesn't help — every record from that key lands on the same partition.
2. **Low-cardinality keys.** Keying by `country_code` with 200 countries and 50 partitions means just a handful of partitions carry most traffic.
3. **Bad partitioner.** Custom partitioner that doesn't spread well, or routes based on a poor predicate (e.g., timestamp bucket).

### Symptoms

- Consumer lag is high on a few partitions, near zero on others.
- Broker disk I/O / network is unbalanced across leaders.
- p99 end-to-end latency is bad even though average is fine.
- Adding consumers doesn't reduce lag.

Metrics to watch:

```
kafka.server:type=BrokerTopicMetrics,name=BytesInPerSec,topic=*    # per-partition variant
kafka.consumer:type=consumer-fetch-manager-metrics,records-lag-max
```

A "tail/median ratio" of bytes-in per partition > 3 is a smell. > 10 is on fire.

### Detection (CLI)

Per-partition message rate via `kafka-run-class kafka.tools.GetOffsetShell`:

```bash
kafka-run-class.sh kafka.tools.GetOffsetShell \
  --broker-list b1:9092 --topic orders --time -1
# orders:0:1234567
# orders:1:1234890
# orders:2:9876543   <-- hot
```

Sample twice with a delta to get rate. Or pull JMX `MessagesInPerSec` tagged by partition.

### Remediation Patterns

**1. Increase partition count.**
Only helps if key distribution is already good. Won't help with a single dominant key.

```bash
kafka-topics.sh --alter --topic orders --partitions 64
```

Caveat: only *new* keys benefit (old keys hash to old partitions if you didn't reshuffle); ordering guarantees per key are preserved across the change only because hash is stable on key.

**2. Key salting (for a hot tenant).**
Append a salt to the key for the hot tenant only, then aggregate downstream:

```java
String key = isHot(tenantId) ? tenantId + "#" + (counter++ % 8) : tenantId;
```

This splits one logical key across 8 partitions. Downstream must handle out-of-order or use a windowed aggregator.

**3. Custom partitioner with two-level routing.**

```java
public int partition(String topic, Object key, byte[] keyBytes, ...) {
    if (isHotKey(key)) {
        return ThreadLocalRandom.current().nextInt(hotPartitions);
    }
    return Math.abs(Utils.murmur2(keyBytes)) % (totalPartitions - hotPartitions) + hotPartitions;
}
```

Reserve a partition range for hot keys, round-robin within it; cold keys hash into the rest.

**4. Sticky Partitioner / KIP-794 Uniform Sticky.**
For *null-key* records, the default since 2.4 batches many records to one partition until a batch closes, then picks another. This trades per-record fairness for throughput. If your data has no key, just rely on this — don't write a custom partitioner.

**5. Re-key upstream.**
If `customer_id` is hot but `(customer_id, order_id)` would be uniform, re-key in a stream processor:

```
orders --[KStream]--> rekey to order_id --> orders.rekeyed
```

### Tradeoff Summary

| Approach | Preserves per-key ordering | Effort | When to use |
|---------|---------------------------|--------|-------------|
| More partitions | Yes | Low | Skew from low cardinality |
| Salting | No (per logical key) | Medium | One/few hot tenants |
| Custom partitioner | No (for hot keys) | Medium | Stable set of hot keys |
| Re-key upstream | No (changes contract) | High | Schema lets you pick a better key |

> [!NOTE]
> Ordering is the price you pay to fix hot partitions. Decide first whether per-key order matters. For analytics, usually not. For ledgers, almost always.

### Interview Follow-ups

- *"Can you add partitions without losing ordering?"* — For *future* records you cannot guarantee old-key→old-partition. The hash mapping changes. If ordering matters, drain consumers, copy to a new topic with the new partition count via MirrorMaker, switch over.
- *"Why is one consumer at 100% CPU while others idle?"* — Almost always a hot partition.
- *"How does this interact with exactly-once?"* — Transactions don't help with skew; they make it worse (transaction coordinator on the producing partition's broker can become a bottleneck).
