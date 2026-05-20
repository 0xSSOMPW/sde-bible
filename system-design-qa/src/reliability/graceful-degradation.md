# Q: Graceful degradation and load shedding — when failure beats unavailability.

**Answer:**

Under stress, a system can do one of three things:

1. **Maintain full service** (best — only if capacity allows).
2. **Degrade gracefully** (serve a reduced version of the feature).
3. **Crash / hang** (worst — affects everyone).

The pattern: serve a worse experience to many rather than fail a few completely.

### Graceful Degradation Examples

| Original feature | Degraded version |
|------------------|------------------|
| Personalized recommendations | Popular items |
| Live cart total | Last known total |
| Real-time fraud check | Async fraud check (allow now, review later) |
| Full search | Cached / popular search results |
| Rich profile page | Bare-bones with name + avatar |
| ML-ranked feed | Chronological feed |

User sees something. They don't see an error.

### Load Shedding

Reject excess requests at the entry point to protect the system from collapse.

```
healthy:    accept all
degraded:   accept critical, reject low-priority
overloaded: accept only health checks; return 503
```

Mechanism:
- Queue depth check at gateway: if > N, return 503.
- Per-tenant quotas: drop excess from one bad actor.
- Adaptive load shedding: shed proportional to observed latency.

Drop early, drop fast. Letting requests queue makes everyone slow.

### Tiered Response

Categorize requests:

- **Critical**: user-paying actions, security, authentication. Never shed.
- **Important**: core feature use. Shed last.
- **Optional**: analytics, marketing, recommendations. Shed first.

Tag at the gateway / load balancer.

### Adaptive Concurrency

Borrowed from TCP: AIMD (Additive Increase, Multiplicative Decrease).

```
in-flight requests = N
observe response time
if RT good: N += 1
if RT bad:  N = N × 0.7
```

Auto-tunes capacity per downstream. Used by Netflix concurrency-limits, Envoy adaptive concurrency.

### Failure Modes for Dependencies

When dependency X fails:

| Strategy | Description |
|----------|-------------|
| Fallback to cache | Return last-known-good |
| Fallback to default | Return safe / empty value |
| Async deferral | Queue, retry later |
| Disable feature | Render UI without the dep |
| Hard fail | Only when no fallback makes sense |

Circuit breaker + fallback is the typical implementation.

### Cache-as-Fallback

```python
def get_recommendations(user):
    try:
        return ml_service.recommend(user)
    except ServiceUnavailable:
        return popular_items_cache.get()
```

When ML service is broken, return popular items instead. UX slightly worse; service still works.

### "Read-Only Mode"

For DB writes: when primary unhealthy, serve reads from replica; reject writes with clear UX.

```
GET /users/me            → works (replica)
POST /orders             → 503 "we're temporarily unable to take new orders, please retry"
```

Better than crashing.

### Feature Flags as Kill Switches

Per-feature toggles let you disable a broken feature without redeploying:

```
if flag("recommendations_enabled"):
    show_recs(user)
```

Flip the flag → recommendations disabled across all instances within seconds.

Used in incidents to isolate damage.

### Backpressure as Degradation

When a downstream is slow, applying backpressure to clients (return 429 with `Retry-After`) is degradation:

```
client should retry later
service stays alive
```

See [Backpressure](../building-blocks/backpressure.md).

### Communicating Degradation

Don't pretend everything's fine:

```
401 → "you're not logged in"
503 → "high traffic, retry in 30 seconds"
"Some features temporarily unavailable" banner
```

Users tolerate degradation if you tell them. They don't tolerate mystery slowness.

### Detection / Triggers

- Latency > threshold.
- Error rate > threshold.
- Queue depth > N.
- Downstream circuit breaker open.
- Manual override (oncall flips a flag).

Automated triggers good for fast response; manual triggers for cases that need judgment.

### Recovery

When pressure subsides, re-enable degraded features:

- Slow ramp (canary back to full).
- Test each tier as it comes back.
- Don't restore instantly — that often re-triggers the original issue.

### Common Mistakes

| Mistake | Reality |
|---------|---------|
| Hard fail on dependency outage | Unnecessary user-visible breakage |
| Cache forever as "fallback" | Stale data masquerading as live |
| No fallback decision made until incident | Engineers panic-decide; bad outcomes |
| Degraded mode rarely tested | Doesn't work when needed |
| Pretending it's fine | Users lose trust |

> [!NOTE]
> Design degraded modes during peace time, not during incidents. The question to ask of every dependency: "if this is down, what does the user see?" If the answer is "an error," design something better.

### Interview Follow-ups

- *"How do you decide between failing and degrading?"* — If the action is critical (payment), fail clearly. If it's auxiliary (recs), degrade silently.
- *"How do you test graceful degradation?"* — Game days; chaos engineering injects dependency failures and verifies UX.
- *"How do you avoid 'always degraded' state?"* — Alert on degradation duration; force a postmortem if a fallback fires for more than N minutes.
