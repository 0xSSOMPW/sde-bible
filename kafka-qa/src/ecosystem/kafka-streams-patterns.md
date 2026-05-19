# Q: What are the core Kafka Streams patterns — KStream, KTable, joins, windowing?

**Answer:**

Kafka Streams is a **library** (not a cluster) for building stateful event-processing apps. The hard part is the conceptual model: stream vs table duality, time, joins, and how state is stored and recovered.

### Stream vs Table Duality

```
KStream<K, V>     KTable<K, V>
record-by-record  changelog of state per key
"events"          "current value per key"
```

- **KStream**: every record is independent. Appending logs.
- **KTable**: per-key, latest value wins. Conceptually a materialized view of a compacted topic.

Same topic can be read as either:

```java
KStream<String, Order> orders = builder.stream("orders");
KTable<String, Order>  latestOrders = builder.table("orders");
```

Convert:

```java
KTable<String, Long> counts = stream
    .groupByKey()
    .count();             // KStream → KTable via aggregation

KStream<String, Order> changes = latestOrders.toStream();   // KTable → KStream of changes
```

### Topology

```
builder.stream("input")
       .filter((k, v) -> v.amount > 0)
       .mapValues(v -> enrich(v))
       .to("output");
```

Every operator is a node in a topology DAG. The runtime executes the topology across **N stream threads** in your app instance, with state stored in local RocksDB.

### State Stores

Aggregations and joins need state. Streams keeps state in **RocksDB** on local disk and backs it up to a **changelog topic** in Kafka.

```
KTable<String, Long> counts = stream.groupByKey().count(
    Materialized.<String, Long, KeyValueStore<Bytes, byte[]>>as("counts-store"));
```

On crash, a new instance restores state by reading the changelog from offset 0 (with the help of standby replicas if configured).

### Joins

Three join modes:

**1. KStream-KStream join** (windowed).

```java
KStream<String, OrderShipped> joined = orders.join(
    shipments,
    (o, s) -> new OrderShipped(o, s),
    JoinWindows.ofTimeDifferenceWithNoGrace(Duration.ofMinutes(5))
);
```

Records within 5 minutes of each other (by **event time**) are joined. Without a window, the join couldn't terminate.

**2. KStream-KTable join** (lookup).

```java
KStream<String, EnrichedOrder> enriched = orders.join(
    customers,                                    // KTable
    (order, customer) -> enrich(order, customer)
);
```

For each order, look up the current customer record. No window. Updates to the table see *future* orders enriched with new values.

**3. KTable-KTable join** (relational).

```java
KTable<String, View> view = users.join(profiles, (u, p) -> new View(u, p));
```

Like a SQL inner join. Changes on either side update the result.

Left, outer, and inner variants exist for each.

### Co-Partitioning Requirement

KStream-KStream and KStream-KTable joins require **co-partitioning**:
- Same number of partitions.
- Same key.
- Same partitioner.

If the input topics aren't co-partitioned, you'll see runtime exceptions. Fix: `repartition()` (creates an internal repartition topic with the right key/partition count).

```java
orders.selectKey((k, v) -> v.customerId)
      .repartition(Repartitioned.as("orders-by-customer").withNumberOfPartitions(12))
      .join(customers, ...);
```

### Time Semantics

Kafka Streams distinguishes:

- **Event time**: timestamp inside the record (when the event happened).
- **Processing time**: when the app sees the record.
- **Ingestion time**: when the broker received it.

Default: event time, extracted from the record's `timestamp` field. Change with a `TimestampExtractor`.

### Windowing

```java
// Tumbling: fixed-size, non-overlapping
TimeWindows.ofSizeWithNoGrace(Duration.ofMinutes(1));

// Hopping: fixed-size, overlap
TimeWindows.ofSizeAndGrace(Duration.ofMinutes(5), Duration.ofMinutes(1))
           .advanceBy(Duration.ofMinutes(1));

// Session: gap-based
SessionWindows.ofInactivityGapWithNoGrace(Duration.ofMinutes(30));

// Sliding: window slides per event
SlidingWindows.ofTimeDifferenceWithNoGrace(Duration.ofMinutes(5));
```

Pattern:

```java
KTable<Windowed<String>, Long> counts = stream
    .groupByKey()
    .windowedBy(TimeWindows.ofSizeWithNoGrace(Duration.ofMinutes(1)))
    .count();
```

The key becomes `Windowed<String>` — original key plus window bounds.

### Grace Period and Late Events

A late event for a window that already closed normally gets dropped. The **grace period** keeps the window open longer:

```java
TimeWindows.ofSizeAndGrace(Duration.ofMinutes(5), Duration.ofMinutes(2))
```

Trade: lower grace = faster window close; higher grace = handles out-of-order events at memory cost.

### Suppress (Emit Once Per Window)

By default, aggregations emit on **every update** — many intermediate values per window.

```java
.suppress(Suppressed.untilWindowCloses(unbounded()))
```

Emits one result per window after it closes. Useful for downstream consumers that don't want noise.

### Interactive Queries

You can query state stores directly from your app:

```java
ReadOnlyKeyValueStore<String, Long> store = streams.store(
    StoreQueryParameters.fromNameAndType("counts-store", QueryableStoreTypes.keyValueStore()));
Long c = store.get("user-1");
```

Useful for serving low-latency reads from materialized state without going through a database.

### Exactly-Once

```java
props.put(StreamsConfig.PROCESSING_GUARANTEE_CONFIG, StreamsConfig.EXACTLY_ONCE_V2);
```

Wraps consume → process → produce → commit in a Kafka transaction. Each input record contributes to outputs atomically. EOS_V2 (Kafka 2.5+) is much cheaper than V1 — one producer per stream thread, not per partition.

### Common Patterns

**1. Event Enrichment.**

```
KStream(orders) JOIN KTable(customers) → KStream(enriched-orders)
```

**2. Real-Time Counters.**

```
KStream(clicks) → groupByKey → windowed-count → KTable(clicks-per-min)
```

**3. Sessionization.**

```
KStream(events) → groupByKey → session-window → KTable(user-sessions)
```

**4. Materialized View / CQRS.**

```
KStream(domain-events) → fold/aggregate → KTable(view) → interactive queries
```

### Common Mistakes

| Mistake | Fix |
|---------|-----|
| Two non-co-partitioned topics joined directly | `repartition()` first |
| Big aggregation state without changelog topic compacted | Disk fills up; ensure changelog is `cleanup.policy=compact` |
| Holding mutable state in processor instances | State must live in state stores, not POJOs |
| Late event dropping silently | Set grace period; route to side topic for late handling |
| Tumbling window on processing time | Use event time for replay-correctness |
| Multiple instances, no standby replicas, slow recovery | `num.standby.replicas: 1` for warm spares |

### Streams vs ksqlDB vs Flink

| Tool | Trade |
|------|-------|
| Kafka Streams | Library — embed in any JVM app. State on local disk + changelog topic. |
| ksqlDB | SQL on top of Streams; lower-code option |
| Apache Flink | Cluster runtime; better for very large state, exactly-once across non-Kafka sinks |

> [!NOTE]
> Kafka Streams is great when your inputs and outputs are Kafka, your state fits on the app instances, and you want zero external coordinators. For multi-source pipelines or massive state, Flink is the better tool.

### Interview Follow-ups

- *"What's the difference between `aggregate` and `reduce`?"* — `reduce` requires the same type in/out. `aggregate` lets the result type differ from the input.
- *"How are stream tasks scheduled?"* — One task per input partition. Threads run multiple tasks. Adding instances rebalances tasks across instances.
- *"What happens to state on rolling deploy?"* — RocksDB stays on disk; instance picks up where it left off. Standby replicas keep warm state on other instances for fast failover.
