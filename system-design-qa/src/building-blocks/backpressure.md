# Q: Backpressure — what is it, and how do you implement it?

**Answer:**

Backpressure is the feedback signal from a slow consumer telling its producer to slow down. Without it, producers overwhelm consumers, queues balloon, latency spikes, and the system collapses. Every streaming and async system needs an explicit backpressure story.

### The Failure Mode

```
producer → unbounded queue → consumer (slow)

Queue grows: 1k, 10k, 100k, 1M messages.
Memory pressure builds. GC churns.
Latency for old messages goes from ms to minutes.
Eventually: OOM, or consumer can never catch up.
```

### The Fix: Apply Backpressure

The producer must respond to the consumer's pace. Three flavors:

**1. Blocking.**
Producer's `send()` blocks until consumer has capacity.

```
bounded queue (capacity=1000)
   send() blocks if full
   recv() unblocks send when capacity returns
```

Simple. Trade: blocked producer threads.

**2. Lossy.**
Drop messages when full.

```
try_send(msg):
  if queue.size < capacity:
    queue.add(msg)
  else:
    drop or sample
```

Used for telemetry, analytics where freshness > completeness.

**3. Reactive (pull-based).**
Consumer signals demand; producer sends only what's requested.

```
consumer.request(10)   → producer sends 10 items
consumer.request(5)    → producer sends 5 more
```

Used by Reactive Streams (Java/JVM), RxJS, Akka Streams.

### Where Backpressure Lives Per Tech

| Layer | Backpressure mechanism |
|-------|------------------------|
| TCP | Built-in window-based flow control |
| HTTP/1.1 | TCP-level only |
| HTTP/2 | Per-stream window + flow control |
| gRPC | Built on HTTP/2; flow control implicit |
| Kafka | Consumer pulls — natural backpressure |
| RabbitMQ | Per-channel flow control; broker can block publishers |
| SQS | Producer never blocked; messages buffer in broker |
| Reactive Streams | `request(n)` method |
| Tokio channels | Bounded `send().await` blocks |

### Application-Level Patterns

**1. Bounded buffer.**

```
queue.put(msg)        # blocks if full
```

Java's `BlockingQueue` capacity-bounded. Producer naturally throttles.

**2. Semaphore for concurrency control.**

```
sem = Semaphore(100)
for task in tasks:
    sem.acquire()
    async_process(task).then(sem.release)
```

Caps in-flight work; doesn't let it spike unboundedly.

**3. Token bucket producer.**

Producer holds a bucket of tokens, replenished at consumer's confirmed pace. Sends only with a token.

**4. Adaptive rate limit.**

Producer measures consumer's response latency; if latency rises, reduce send rate. AIMD (Additive Increase, Multiplicative Decrease) — same as TCP congestion control.

### When Backpressure Isn't Enough — Shed Load

If you can't slow down, drop messages. Patterns:

- **Tail drop**: drop new messages when full.
- **Head drop**: drop oldest (newest data more valuable, e.g., telemetry).
- **Priority drop**: drop low-priority first.
- **Random sampling**: drop 1 in N to maintain a representative sample.

Better than collapsing. Document what's dropped.

### Async / Reactive Backpressure

Reactive Streams spec defines a Publisher-Subscriber protocol with `request(n)`:

```java
Flux.range(1, 1000)
    .onBackpressureBuffer(100)       // buffer up to 100
    .onBackpressureDrop(msg -> log("dropped: " + msg))
    .subscribe(subscriber);
```

Operators like `buffer`, `drop`, `latest` define overflow strategy.

### Buffer Sizing

```
buffer_size = expected_lag × throughput
```

If consumer might lag by 1s and you process 10k/sec, buffer of 10k handles 1s spike.

Larger buffers smooth bursts; smaller buffers reveal problems faster. Tune for your SLA.

### Backpressure vs Throttling

- **Backpressure**: feedback from consumer to producer.
- **Throttling / rate limiting**: producer caps itself (or is capped by gateway).

Both reduce flow; backpressure is reactive (responds to actual capacity); throttling is proactive (predefined limit).

### Multi-Layer Backpressure

```
client ─── HTTP/2 flow control ──► gateway
                                     │
                                     │ bounded channel
                                     ▼
                                  service
                                     │
                                     │ async submit with semaphore
                                     ▼
                                  worker pool
                                     │
                                     │ bounded queue
                                     ▼
                                  downstream
```

Each layer enforces capacity. The slowest layer signals back through every prior layer.

### Common Mistakes

| Mistake | Result |
|---------|--------|
| Unbounded queue / channel | OOM when consumer slow |
| Async submit with no concurrency cap | Thousands of in-flight requests crush downstream |
| Catch-and-ignore overflow | Silent loss |
| Logging at INFO every dropped message | Log floods worse than the original problem |
| No metric for buffer size / age | Can't see backpressure happening |

### Observability

- `queue_size` (gauge).
- `queue_oldest_age` (oldest pending item's age).
- `producer_blocked_total` (counter).
- `dropped_messages_total` (counter, with reason label).

Alerting on these surfaces backpressure before users feel it.

> [!NOTE]
> Backpressure is not optional. Every async boundary in a system needs an explicit policy: bound the buffer, define overflow, surface the metric. "Just make the queue bigger" is how systems die.

### Interview Follow-ups

- *"How does Kafka apply backpressure?"* — Consumer pulls. If consumer can't keep up, lag grows; producer keeps producing into the log. Backpressure to producer happens only if log retention is exceeded (very long lag) — by then it's a real outage.
- *"What about WebSocket backpressure?"* — Per-connection send buffer; if app can't drain client's slow socket, write to socket blocks; either drop or disconnect slow client.
- *"How does AIMD relate?"* — TCP congestion control mimics what application backpressure should do — slowly raise rate, halve it on signs of trouble.
