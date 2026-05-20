# Q: Circuit breaker, bulkhead, timeout — failure-isolation patterns.

**Answer:**

Three patterns from the **fault-tolerance toolbox**, each protecting different things:

- **Timeout**: bound how long you wait.
- **Circuit breaker**: stop calling a failing dependency.
- **Bulkhead**: isolate resources so one failure doesn't sink everything.

### Timeout

The simplest, most under-used pattern.

```
client.get(url, timeout=2s)
db.query(sql, timeout=500ms)
```

Without timeouts:
- Slow downstream stalls your threads.
- Threads exhaust → can't serve any request.
- Cascading failure.

Pick timeouts by:
- Median + 3× standard deviation of normal.
- p99 of healthy + margin.
- Aggressive enough that a stuck downstream doesn't wait forever.

Default: 1–3 seconds for HTTP; 100–500 ms for in-DC RPC.

### Circuit Breaker

State machine:

```
CLOSED (normal):
  pass calls through; track success/failure ratio
  if failure rate > threshold → OPEN

OPEN:
  fail fast — return error without calling
  after wait period → HALF_OPEN

HALF_OPEN:
  pass a few probe calls
  all succeed → CLOSED
  any fail → OPEN
```

Why: when downstream is broken, retrying is wasteful AND amplifies the outage. Stop calling.

Library implementations: Hystrix (deprecated), resilience4j (JVM), polly (.NET), sony/gobreaker (Go), opossum (Node).

Configuration:
- `failure_threshold`: 50% over last N calls.
- `sliding_window`: e.g., 100 calls or 10 seconds.
- `open_duration`: 30 seconds typical.
- `half_open_probes`: 5 calls.

### Bulkhead

Borrowed from ship design: walls between sections so flooding one doesn't sink the ship.

In services: isolate resources per dependency.

```
threadpool[orders]:    100 threads
threadpool[catalog]:    50 threads
threadpool[email]:      20 threads
```

Catalog goes slow → only catalog threads block; orders thread pool unaffected.

Without bulkheads, all calls share one pool. Catalog slowness starves orders.

Variants:
- **Thread pool per dependency** (Hystrix style).
- **Semaphore per dependency** (lighter; no thread overhead).
- **Connection pool per dependency** (DB, HTTP).

### Combined Pattern

```
client → timeout → circuit breaker → bulkhead → call dependency
```

1. Bulkhead caps concurrency.
2. Circuit breaker fails fast when dependency unhealthy.
3. Timeout bounds individual call duration.

All three together = isolation depth.

### Implementation Example (Resilience4j)

```java
@CircuitBreaker(name = "paymentsApi", fallbackMethod = "fallback")
@TimeLimiter(name = "paymentsApi")
@Bulkhead(name = "paymentsApi")
@Retry(name = "paymentsApi")
public CompletableFuture<PaymentResp> charge(ChargeReq req) {
    return client.post(...);
}

PaymentResp fallback(ChargeReq req, Throwable t) {
    return PaymentResp.queued(req);
}
```

Order matters: outer to inner is Retry → Breaker → RateLimiter → Bulkhead → TimeLimiter.

### Observability

- **CB state**: per dependency, time spent in each state.
- **Bulkhead saturation**: in-use vs max permits.
- **Timeout rate**: how often timeouts fire.

Alert on:
- CB in OPEN state for > N minutes.
- Bulkhead near saturation.
- Timeout rate > threshold.

### Choosing Thresholds

Too tight: false-positive opens during normal load spikes.
Too loose: opens too late, damage done.

Start at 50% failure rate over a meaningful window (50–100 calls). Tune from data.

### Common Mistakes

| Mistake | Reality |
|---------|---------|
| No timeout (uses library default) | Default is often "wait forever" |
| Single shared thread pool | Bulkhead missing; one bad dep takes everything |
| Circuit breaker on 4xx errors | 4xx is the caller's bug; doesn't indicate downstream broken |
| Open CB without fallback | Just returns errors; might as well have failed slowly |
| Same CB for hot path + background | Open CB on background work kills user-facing requests |

### When Not To Use Each

**Timeout**: always use. There's no excuse.

**Circuit breaker**: skip for non-critical path that you'd retry anyway. Don't break on rare network blips.

**Bulkhead**: overkill for monoliths with a single dependency. Required for services calling many.

### Pattern Composition for Production Service

```
Per dependency:
  Bulkhead: 50 concurrent permits
  Timeout: 2 seconds
  Circuit breaker: 50% failure over 100 calls / 30s window
  Retry: 3 attempts with jitter (only on idempotent)
  Fallback: cached response or sensible default
```

Track each layer's metrics.

> [!NOTE]
> These patterns aren't magic — they protect against the *specific failure* of "downstream slow / failing." For other failures (bug in your service, data corruption), they don't help. Mix with health checks, runbooks, capacity planning.

### Interview Follow-ups

- *"What's the difference between a circuit breaker and a rate limiter?"* — Breaker reacts to failure rate. Limiter caps QPS regardless of success. Different protections, often used together.
- *"How does a circuit breaker know when to close?"* — HALF_OPEN state allows a few probes; success → CLOSED; failure → back to OPEN.
- *"What's load shedding?"* — Service deliberately rejects requests when overloaded (often based on queue size or RT). See [Graceful Degradation](./graceful-degradation.md).
