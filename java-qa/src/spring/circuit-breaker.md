# Q: What is a circuit breaker, and how do you use Resilience4j in Spring Boot?

**Answer:**

A circuit breaker is a pattern that **stops calling a failing dependency** to give it time to recover and to prevent cascading failure. Resilience4j is the modern Java implementation (Hystrix has been in maintenance since 2018).

### The Failure It Prevents

```
Service A ──► Service B (slow / down)

Without breaker:
  - A's threads block waiting on B.
  - Connection pool exhausts.
  - A starts failing requests it could otherwise serve.
  - Cascade: any upstream of A starts to fail.
```

A circuit breaker stops calling B after a threshold of failures, returning a fallback immediately. A's resources stay healthy. B gets a chance to recover.

### The State Machine

```
        ┌────────────┐  failure rate > threshold   ┌───────────┐
        │   CLOSED   │ ─────────────────────────►  │   OPEN    │
        │ (calls B)  │                             │ (fails    │
        │            │  ◄─────────────────────     │  fast)    │
        └────────────┘   success in HALF_OPEN      └─────┬─────┘
                                                          │ wait duration
                                                          ▼
                                                  ┌───────────────┐
                                                  │   HALF_OPEN   │
                                                  │ (probe calls) │
                                                  └───────────────┘
```

- **CLOSED**: normal operation; track success/failure metrics in a sliding window.
- **OPEN**: requests fail fast (don't even try B). After a wait, transition to half-open.
- **HALF_OPEN**: allow N probe calls. If they succeed, close. If they fail, open again.

### Resilience4j Setup

```xml
<dependency>
    <groupId>io.github.resilience4j</groupId>
    <artifactId>resilience4j-spring-boot3</artifactId>
</dependency>
```

```yaml
resilience4j:
  circuitbreaker:
    instances:
      paymentsApi:
        sliding-window-type: COUNT_BASED
        sliding-window-size: 50
        minimum-number-of-calls: 20
        failure-rate-threshold: 50           # percent
        slow-call-rate-threshold: 60
        slow-call-duration-threshold: 2s
        wait-duration-in-open-state: 30s
        permitted-number-of-calls-in-half-open-state: 5
        automatic-transition-from-open-to-half-open-enabled: true
```

Annotate the call:

```java
@Service
class PaymentClient {

    @CircuitBreaker(name = "paymentsApi", fallbackMethod = "fallback")
    public PaymentResp charge(ChargeReq req) {
        return restClient.post().uri("/charges").body(req).retrieve().body(PaymentResp.class);
    }

    PaymentResp fallback(ChargeReq req, Throwable t) {
        return PaymentResp.queued(req);   // or throw a domain exception
    }
}
```

The fallback **must have the same signature** plus a trailing `Throwable` parameter (or specific exception type).

### Combine with Other Patterns

A circuit breaker alone is rarely enough. Stack:

```java
@Retry(name = "paymentsApi")
@CircuitBreaker(name = "paymentsApi", fallbackMethod = "fallback")
@Bulkhead(name = "paymentsApi")
@TimeLimiter(name = "paymentsApi")    // for CompletableFuture-returning methods
public CompletableFuture<PaymentResp> charge(ChargeReq req) { ... }
```

Order matters (outermost to innermost as decorator):

```
Retry → CircuitBreaker → RateLimiter → Bulkhead → TimeLimiter
```

- **TimeLimiter**: bounds individual call duration (slow B doesn't tie up a thread).
- **Bulkhead**: bounds concurrent calls — isolates B's failure from A's other dependencies.
- **RateLimiter**: caps QPS to be a good citizen.
- **Retry**: handles transient failures with backoff.
- **CircuitBreaker**: handles sustained failure.

### Choosing Thresholds

| Setting | Typical | Why |
|---------|---------|-----|
| `minimum-number-of-calls` | 20–50 | Avoid flipping on a tiny sample |
| `failure-rate-threshold` | 50% | Half the calls failing = something's wrong |
| `slow-call-duration-threshold` | p99 of healthy traffic × 2 | Don't conflate slow with broken |
| `wait-duration-in-open-state` | 30s | Long enough for dependency to recover, short enough to retry quickly |
| `permitted-number-of-calls-in-half-open-state` | 5–10 | Probe load |

### What Counts as a Failure

By default: any thrown exception. Customize:

```yaml
record-exceptions:
  - org.springframework.web.client.RestClientException
  - java.io.IOException
ignore-exceptions:
  - com.example.ValidationException     # 4xx is the client's fault, don't open breaker
```

Critical: do not let a 400-class response open the breaker. That's the *caller's* bug, not the dependency's.

### Observability

Resilience4j publishes Micrometer metrics:

```
resilience4j.circuitbreaker.state{name="paymentsApi", state="open"} 1
resilience4j.circuitbreaker.calls{name="paymentsApi", kind="successful"} 12345
resilience4j.circuitbreaker.failure.rate{name="paymentsApi"} 0.42
```

Alert on:
- `state == open` for more than a brief flap.
- `failure.rate > threshold * 0.8` (about to open).

Also enable health indicator:

```yaml
management.health.circuitbreakers.enabled: true
```

### Common Mistakes

| Mistake | Fix |
|---------|-----|
| Catching the exception inside the annotated method | Breaker never sees the failure — let it propagate |
| Fallback that *also* calls the dependency | Fallback must be cheap and local |
| Same `name` for unrelated dependencies | One bad dependency opens the breaker for both |
| Counting 404 as a failure | Use `ignore-exceptions` for expected client-side errors |
| Wrapping a `@Cacheable` method directly | Cache-hit short-circuits the breaker; usually fine but be aware |

> [!NOTE]
> A circuit breaker is **not** a load shedder. If you're overloading B, breakers will hide it temporarily but not solve it. Pair with rate limiters and capacity planning.

### Interview Follow-ups

- *"Why not just retry?"* — Retries multiply load on a struggling dependency. A breaker stops calling at all.
- *"Difference between bulkhead and circuit breaker?"* — Bulkhead limits *concurrent* calls to isolate failure domains. Breaker decides *whether* to call at all based on recent failure rate. Complementary.
- *"How is this related to a sliding-window rate limiter?"* — Same data structure (sliding window), different decision: rate limiter caps QPS; breaker caps failure rate.
