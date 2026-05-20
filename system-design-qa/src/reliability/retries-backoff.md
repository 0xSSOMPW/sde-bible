# Q: Retries, backoff, jitter — done right.

**Answer:**

Retrying failed requests is essential. Done naively, retries amplify outages — the "retry storm." The recipe: exponential backoff + jitter + cap + circuit breaker.

### What to Retry

| Failure | Retry? |
|---------|-------|
| Network timeout | Yes |
| 5xx response | Yes (probably transient) |
| 429 (rate limited) | Yes, respect `Retry-After` |
| 503 (service unavailable) | Yes, respect `Retry-After` |
| 4xx (400, 401, 404, 422) | No (client error) |
| Connection refused | Yes |

Retry **idempotent** operations only. Non-idempotent: include `Idempotency-Key`.

### Naive Retry (Don't)

```
while not success:
    response = call()
    if 5xx: continue       # immediate retry
```

If 100 clients all retry immediately, the server gets 100× load instantly. The thing that caused the failure (overload) gets worse.

### Exponential Backoff

```
attempt 1: wait 100 ms
attempt 2: wait 200 ms
attempt 3: wait 400 ms
attempt 4: wait 800 ms
...
attempt N: wait min(2^N × base, max_backoff)
```

Spreads retries over time. Reduces aggregate load.

### Add Jitter

Without jitter, all clients backoff identically and retry at the same instant:

```
delay = 2^attempt × base
```

Becomes:

```
delay = random(0, 2^attempt × base)         # full jitter
```

Or AWS's recommendation:

```
delay = min(max_backoff, random(base, 2^attempt × base))    # decorrelated jitter
```

Spreads retries across time; eliminates synchronized retry storms.

### The Full Recipe

```python
def call_with_retry(func, max_attempts=5, base=0.1, max_backoff=10):
    for attempt in range(max_attempts):
        try:
            return func()
        except RetryableException as e:
            if attempt == max_attempts - 1:
                raise
            delay = min(max_backoff, base * 2**attempt * random())
            time.sleep(delay)
```

### Respect `Retry-After`

```http
HTTP/1.1 429 Too Many Requests
Retry-After: 30
```

Server tells you when to come back. Honor it.

```python
if response.status == 429:
    wait_sec = int(response.headers.get("Retry-After", 1))
    sleep(wait_sec)
```

### Cap the Retries

Infinite retries → resource leak (connections, threads) when downstream is broken.

```
max_attempts = 5    # typical for synchronous user-facing requests
max_attempts = 20+  # for background jobs
```

After cap, propagate failure upward. Don't pretend it succeeded.

### Retry at Which Layer?

Multiple retry layers stack and amplify:

```
client retry × LB retry × app retry × HTTP client retry = 16+ attempts per user click
```

Bad. Pick **one layer** to retry. Other layers should fail fast.

Typical: HTTP client library retries (with reasonable defaults); other layers don't.

### Circuit Breaker

After many failures in a row, **stop calling** for a while:

```
state: CLOSED (calls allowed) → OPEN (calls fail fast) → HALF_OPEN (probe a few)
```

See [Circuit Breaker](./circuit-breaker-bulkhead.md).

Pair with retry: retry helps transient failures; circuit breaker handles sustained failure.

### Hedged Requests

For latency optimization, not failure handling:

```
t=0: send request to A
t=p95: still no response → send to B
t=any: take first response, cancel other
```

Used by Google for tail-latency reduction. Doubles load on slow operations.

### Retry Storm

Classic outage pattern:

```
Service A is slow.
Clients of A retry.
A gets 3× normal load.
A becomes slower / starts failing.
Clients retry more.
A is now totally overwhelmed.
```

Mitigations:
- Aggressive timeouts (fail fast).
- Limited retries.
- Circuit breaker.
- Backoff + jitter.
- Load shedding by A itself.

### Retries vs Backoff in Real Systems

```
HTTP client (e.g., reqwest, axios):
  retries: 3
  backoff: exp with jitter
  max delay: 5s

Async queue (Kafka consumer):
  retries: infinite (or N to DLQ)
  backoff: exp with cap 5 min

DB driver:
  retries: 2-3
  on serialization-failure: retry with backoff
```

### Idempotency Is Mandatory

Retries that aren't idempotent corrupt state.

For HTTP POST: `Idempotency-Key` header → server dedups.
For DB writes: unique constraint on natural key.
For messages: dedup by `event_id`.

See [Idempotency](./idempotency.md).

### Common Mistakes

| Mistake | Reality |
|---------|---------|
| Retry immediately (no backoff) | Storm amplifies outage |
| Same delay every attempt | Synchronized storm |
| No max attempts | Resource leak |
| Retry POSTs without idempotency key | Double-charges, duplicate orders |
| Multiple layers retrying | Multiplies load |
| Retry on 4xx | Will never succeed |

> [!NOTE]
> The most operationally important retry pattern is "exponential backoff with full jitter." Memorize it; apply it everywhere. The default in libraries is usually NOT this — set it yourself.

### Interview Follow-ups

- *"What's the difference between full and decorrelated jitter?"* — Full jitter samples uniformly over the backoff window. Decorrelated jitter widens with each attempt; better avoids synchronized retries.
- *"How do you handle retries with stream processors?"* — Tiered retry topics (Kafka) — failed events flow into delay topics; eventually DLQ.
- *"What's a 'retry budget'?"* — Cap on the rate of retries the system as a whole tolerates. If retries exceed N% of total traffic, stop retrying. Used by Envoy.
