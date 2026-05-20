# Q: Latency vs throughput vs tail latency — why does p99 matter more than average?

**Answer:**

Two systems can have the same average latency and feel completely different to users. The difference is the **distribution**. Modern system design is dominated by the long tail (p99, p99.9), not by the mean.

### Definitions

- **Latency**: time for one request, end to end.
- **Throughput**: requests served per unit time.
- **Bandwidth**: bytes per unit time.

They aren't independent: by Little's Law, `concurrent_requests = throughput × latency`. If you fix concurrency, raising one drops the other.

### Why Averages Lie

```
Request latencies (ms):
  20, 25, 22, 30, 27, 25, 28, 1200, 22, 26
                                ^
  Average: 142 ms                slow tail
  Median:  26 ms
  p99:    1200 ms
```

Average gets dragged by a single outlier. Median + percentiles tell the real story.

### Percentiles Cheat Sheet

| Percentile | Meaning |
|------------|---------|
| p50 (median) | Half of users see this or better |
| p90 | 1 in 10 users sees worse |
| p99 | 1 in 100 sees worse |
| p99.9 | 1 in 1000 sees worse |
| p99.99 | 1 in 10,000 — the "tail of the tail" |

### Fan-Out Multiplies Tail Latency

A page that issues N parallel calls and waits for all returns when the **slowest** one returns. If each call has 1% chance of being slow:

```
N = 1:    1% of requests slow
N = 10:   1 - 0.99^10  ≈ 10% slow
N = 100:  1 - 0.99^100 ≈ 63% slow!
```

p99 of the **service** becomes p50 of the **page**. This is why backend latency p99 matters more than average — fan-out amplifies it.

### Sources of Tail Latency

- **GC pauses** (long for big heaps, short with ZGC).
- **JIT warmup** on cold instances.
- **Cache misses** to slower storage.
- **Lock contention** under load.
- **Network jitter / packet loss → retransmit**.
- **Disk fsync / log rotation**.
- **Cold connections** (TCP slow start, TLS handshake).
- **Coordinated omission**: load generator stops counting when it can't push, hiding the real tail.

### Latency Numbers to Remember

```
DC round-trip:           ~500 µs
Cross-AZ:                ~1–2 ms
Cross-region (US ↔ EU):  ~80–100 ms
Cross-continent:         ~150 ms
Memory read:             ~100 ns
SSD random read:         ~100 µs
HDD seek:                ~10 ms
```

A simple "fetch from DB" query in another region is automatically ~100 ms before you do any work.

### Throughput Tuning

Increase throughput via:

- **Parallelism** (multiple threads/cores).
- **Pipelining** (overlap requests; multiple in flight per connection).
- **Batching** (Kafka producer linger, DB multi-row insert).
- **Async I/O** (don't block threads on remote calls).
- **Avoid contention** (locks, hot rows, GC).

Each often hurts latency for individual requests in exchange for system-wide throughput.

### Latency Tuning

Reduce latency via:

- **Cache** (RAM-fast vs disk/network).
- **Move compute to data** (e.g., DB-side aggregation).
- **Reduce serial dependencies** (parallelize independent calls).
- **Reduce hops** (CDN, edge compute).
- **Connection reuse** (HTTP keep-alive, gRPC streams).
- **Pre-warm** caches and connection pools.

### Throughput vs Latency Trade-off

```
Throughput
   │             ╱╲   ← optimal point
   │           ╱    ╲
   │         ╱        ╲      ← degradation
   │       ╱            ╲
   │     ╱
   │   ╱
   │ ╱
   └──────────────────────────►   Concurrency / load
```

Past the optimal point, adding load **reduces** throughput because queues fill, contention rises, GC churns. The same shape governs every queueing system.

### Load Shedding > Queuing

When a service is overloaded, queueing more requests makes it worse — each new request sits behind a longer queue, increasing latency, until upstream times out and retries. Better: **shed** requests at the door.

```
Healthy:    accept all requests
Degraded:   accept critical, reject low-priority
Overloaded: reject all but health checks, surface clear 503
```

### Hedged Requests

Tail latency mitigation: send a duplicate request to a second replica after a short delay (p95). Take whichever returns first. Cancels the slow one if possible.

Pioneered by Google for search. Cheap insurance against a single slow node.

```
t=0:    send to A
t=p95:  if no response, send to B
t=any:  return first response, cancel the other
```

### Coordinated Omission

A common benchmark bug: load generator can't keep up with target QPS, so it backs off — the slow responses it's measuring don't include the time spent *waiting* to send. Real-world load doesn't get to back off.

Use `wrk2`, `tlb-bench`, or modern tools that compensate. Otherwise your published p99 is fiction.

### Latency Budgets

In an SLA of 200 ms p99 end to end:

```
Browser → CDN:       30 ms     (network)
CDN → API gateway:    5 ms
Gateway → service:    1 ms
Service compute:     50 ms
Service → cache:      2 ms
Cache miss → DB:     20 ms
Serialization:        5 ms
Network back:        30 ms
                    ────
                    143 ms      (within budget)
```

Each layer must fit within its slice. Cache miss on every request blows the budget.

### Common Mistakes

| Mistake | Reality |
|---------|---------|
| Reporting average latency | Hides tail; useless for SLA |
| Optimizing p50 when fan-out exists | Page latency is dominated by tail |
| Adding capacity to fix latency | Helps until queue depths drop; not a cure for slow code |
| Synchronous calls in series | Sums latencies; parallelize independent ones |
| Ignoring connection setup | TLS + DNS can add 100+ ms; pool aggressively |

> [!NOTE]
> The most operationally important number in any service is its p99 latency under realistic load. Track it. Alert on it. Tune it. Average latency is a vanity metric.

### Interview Follow-ups

- *"How do you measure latency under load?"* — Distributed tracing (sample); aggregate via histogram metrics (Prometheus + `histogram_quantile`); load test with `wrk2` or `k6`.
- *"What's a hedged request, and when not to use it?"* — Don't use for non-idempotent writes; don't use when load is already high (doubles work).
- *"Why is p99 a worse SLO than `latency ≤ X for 99% of requests / minute`?"* — Both describe the same thing if framed right; the second form is what alerting actually does — windowed.
