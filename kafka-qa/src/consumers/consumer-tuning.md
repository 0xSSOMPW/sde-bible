# Q: How do you tune a Kafka consumer for throughput, latency, and safety?

**Answer:**

Consumer tuning is mostly the interplay of **four properties**: how much you fetch per poll, how often you poll, how often you commit, and what your session/heartbeat timeouts look like. Misconfiguring any of them produces familiar failures: rebalance storms, duplicate processing, lag spikes, or silent data loss.

### The Poll Loop

```java
while (running) {
    var records = consumer.poll(Duration.ofMillis(500));
    for (var r : records) process(r);
    consumer.commitSync();
}
```

Three timers run during this loop:

- `session.timeout.ms`: heartbeat thread keeps group membership. If no heartbeat within this window, the broker considers the consumer dead.
- `heartbeat.interval.ms`: how often heartbeats are sent. Must be < session.timeout/3.
- `max.poll.interval.ms`: max wall time between two `poll()` calls. If processing takes longer, the broker kicks you out → rebalance.

### Throughput Tuning

| Setting | Default | Increase for throughput |
|---------|---------|-------------------------|
| `fetch.min.bytes` | 1 | 10 KB — broker batches more before responding |
| `fetch.max.wait.ms` | 500 | 500–1000 — pair with above; wait for more data |
| `max.partition.fetch.bytes` | 1 MB | 5 MB — bigger payloads per partition per fetch |
| `max.poll.records` | 500 | 1000–5000 — bigger batches per poll |
| `receive.buffer.bytes` | 64 KB | 1 MB — TCP buffer |

Bigger batches = fewer round trips, fewer commits, more throughput. Trade: higher end-to-end latency, bigger spike on rebalance (have to re-process the in-flight batch).

### Latency Tuning

| Setting | For lower latency |
|---------|-------------------|
| `fetch.min.bytes=1` | Don't wait for batching |
| `fetch.max.wait.ms=10` | Cap wait |
| `max.poll.records=100` | Smaller batches process faster end-to-end |

Latency and throughput pull in opposite directions. Pick the workload's priority.

### `max.poll.interval.ms` — the Most Misunderstood Setting

Default: 5 minutes (Kafka 2.x+). If your processing for one batch takes longer, you're rebalanced.

Common scenarios:
- Heavy per-record work (image processing, ML inference) + big batches → exceed limit.
- External API call inside the loop with long timeout.
- GC pause / app stall.

Fixes (in order):
1. Reduce `max.poll.records` so each batch is smaller.
2. Increase `max.poll.interval.ms` (last resort — masks real problems).
3. Move heavy work to a thread pool, but careful: you still must keep polling to maintain group membership.

### The Background-Processing Pattern

If you must process slowly:

```java
while (running) {
    var records = consumer.poll(Duration.ofMillis(100));
    for (var r : records) workerPool.submit(() -> process(r));
    // Don't commit yet — wait for workers to finish.
}
```

Caveats:
- Lose **ordering** across records in a partition.
- Must track which offsets are safe to commit (only commit offsets whose work is fully done).
- Must `pause()` the partition while workers are saturated (`unsubscribe()` would lose membership).

Better: increase partition count and run more consumers — keep the simple poll loop.

### Offset Commit Strategies

```yaml
enable.auto.commit: false           # production default
auto.commit.interval.ms: 5000       # only used if auto-commit is true
```

**Manual sync commit** (safest):

```java
consumer.commitSync();
```

Blocks until brokers ack. Adds latency but you know offsets are durably committed.

**Manual async commit** (fast):

```java
consumer.commitAsync((offsets, ex) -> {
    if (ex != null) log.warn("commit failed", ex);
});
```

Fire-and-forget. On shutdown, do a final `commitSync()` to ensure latest offsets stick.

**Auto commit** (don't use in production):

```java
enable.auto.commit=true
```

Offsets advance every 5 seconds regardless of whether processing succeeded. Silent data loss / duplicate processing on crash.

### Rebalance Behavior

When a consumer joins/leaves, partitions are reassigned. Use a callback to flush state and commit:

```java
consumer.subscribe(List.of("orders"), new ConsumerRebalanceListener() {
    public void onPartitionsRevoked(Collection<TopicPartition> tp) {
        consumer.commitSync();             // flush before yielding
    }
    public void onPartitionsAssigned(Collection<TopicPartition> tp) {
        // optionally seek
    }
});
```

With `partition.assignment.strategy=CooperativeStickyAssignor` (default Kafka 3+), revocation is *incremental* — only re-assigned partitions are revoked, not all of them. Massively reduces rebalance pain.

### Static Membership

```yaml
group.instance.id: ${HOSTNAME}
session.timeout.ms: 60000
```

Set a persistent instance ID and the broker keeps your slot reserved for `session.timeout.ms` after you disappear. Restart within that window = no rebalance.

Use for stateful consumers (Kafka Streams especially).

### Consumer Lag

Lag = `log end offset - committed offset` per partition. The single most important metric.

```
kafka-consumer-groups.sh --bootstrap-server b1:9092 \
  --group orders-app --describe
# TOPIC      PARTITION  CURRENT-OFFSET  LOG-END-OFFSET  LAG
# orders     0          12345           12345           0
# orders     1          12000           12500           500
```

Alert on:
- `lag` increasing trend (consumer can't keep up).
- `lag > N` absolute threshold per business SLA.
- Lag concentrated on one partition (hot partition).

Per-consumer JMX:

```
kafka.consumer:type=consumer-fetch-manager-metrics,
  client-id=*,name=records-lag-max
```

### `auto.offset.reset` for New Groups

- `latest` (default): start at end of topic. Skip historical data.
- `earliest`: start from beginning. Reprocess everything.
- `none`: throw if no committed offset.

This only applies to **new** consumer groups or expired offsets. Don't expect it to matter on a running app.

### Diagnostic Checklist for Lag Spikes

1. Is one partition lagging more than others? → hot key, look upstream.
2. Is processing CPU-bound? → profile, scale consumers (up to partition count).
3. Is downstream (DB, API) slow? → it's downstream, not Kafka.
4. Is GC pause > heartbeat? → tune G1/ZGC, increase heap.
5. Is `max.poll.interval.ms` getting exceeded? → batch too big.

### Common Mistakes

| Mistake | Reality |
|---------|---------|
| `enable.auto.commit=true` in production | Offsets advance without confirming success — data loss |
| Spawning a thread per record without managing offsets | Out-of-order commits, dupes |
| Setting `session.timeout.ms` very large to avoid rebalance | Real failures take long to detect |
| `max.poll.records=10000` then complaining about rebalance | Decrease records or increase poll interval |
| Sharing one consumer across threads | `KafkaConsumer` is not thread-safe |
| Committing offset of last processed record (not last + 1) | Replays that record after restart |

`commitSync` semantics: commit offset = "next offset to read". Use `record.offset() + 1`.

> [!NOTE]
> The single most useful guideline: poll fast, process bounded, commit explicitly. Almost every "Kafka is unreliable" story traces back to violating one of these three.

### Interview Follow-ups

- *"Why does my consumer lag at the same time every day?"* — Compaction or retention deletes / off-peak production lulls / scheduled batch jobs upstream. Look at producer rate per partition.
- *"`commitSync` vs `commitAsync` — which?"* — `Async` in the hot path; `sync` on shutdown and inside rebalance callback. Together they minimize duplication.
- *"What is `fetch.max.bytes` vs `max.partition.fetch.bytes`?"* — First caps total fetch response. Second caps per-partition contribution. Keep total ≥ partitions × per-partition.
