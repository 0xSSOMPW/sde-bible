# Q: Rate limiting algorithms — token bucket, leaky bucket, fixed/sliding window.

**Answer:**

Rate limiting protects services from overload, abuse, and runaway costs. The four canonical algorithms differ in **burst tolerance** and **memory cost**.

### Token Bucket

A bucket holds up to `capacity` tokens. Tokens added at rate `R` per second. Each request consumes one token. Empty bucket → reject (or queue).

```
state per client: tokens (float), last_refill (timestamp)

on request:
    now = current_time()
    elapsed = now - last_refill
    tokens = min(capacity, tokens + elapsed * R)
    last_refill = now
    if tokens >= 1:
        tokens -= 1
        return allow
    else:
        return reject
```

Properties:
- Allows **bursts** up to `capacity`.
- Average rate over time = `R`.
- Per-client state: two floats. Cheap.

Used by: AWS API Gateway, most production limiters.

### Leaky Bucket

Imagine a bucket with a hole leaking at constant rate. Requests pour in; bucket overflowing → reject.

```
state: water (current depth)

on request:
    now = current_time()
    leak = (now - last_check) * leak_rate
    water = max(0, water - leak)
    if water + 1 <= capacity:
        water += 1
        return allow
    else:
        return reject
```

Properties:
- **Smooth output**: requests processed at constant rate.
- No bursts above leak rate.
- Used when downstream needs constant pace (legacy systems, expensive APIs).

Trade vs token bucket: token bucket allows bursts; leaky bucket forces uniform rate.

### Fixed Window Counter

Count requests per fixed window (per minute):

```
state: counter, window_start

on request:
    if now > window_start + 60:
        counter = 0
        window_start = now
    if counter < limit:
        counter += 1
        return allow
    else:
        return reject
```

Properties:
- Simple, cheap (one counter).
- **Burst at boundary**: limit=100/min; user sends 100 at 12:00:59 + 100 at 12:01:00 → 200 requests in 1 second.
- Discrete window unfairness for clients arriving near boundary.

### Sliding Window Log

Store timestamp of each request; count those in the last N seconds.

```
state: list of timestamps (per client)

on request:
    drop timestamps older than now - window
    if len(timestamps) < limit:
        timestamps.append(now)
        return allow
    else:
        return reject
```

Properties:
- Accurate.
- Memory: O(N) per client (worst case = limit timestamps).
- Cleanest semantics; expensive at scale.

### Sliding Window Counter

Hybrid: weighted blend of two adjacent fixed windows.

```
                                       current request
window N-1:  ████████████              │
window N:           ████████████   ────┘  (you're here)

elapsed in current window = 0.4
count = old_window_count * (1 - 0.4) + current_window_count
```

Properties:
- O(1) memory.
- Smoother than fixed window.
- Approximate but production-good.

Used by Cloudflare, Nginx (`limit_req` with leaky bucket nuance).

### Comparison

| Algorithm | Burst tolerance | Memory | Smoothness | Complexity |
|-----------|----------------|--------|------------|------------|
| Token bucket | Up to `capacity` | O(1) | Medium | Low |
| Leaky bucket | None | O(1) | High | Low |
| Fixed window | Up to `2×limit` at boundary | O(1) | Low | Lowest |
| Sliding log | None | O(limit) | Highest | Medium |
| Sliding counter | Modest | O(1) | High | Medium |

### Where to Limit

```
Client                    Edge / CDN                    Service
  │  per-user, per-key       per-IP, per-ASN              per-tenant, per-route
  ▼                                                       │
┌─────┐    ┌──────────┐    ┌─────────────┐    ┌──────────┴────┐
│ App │ ── │   CDN    │ ── │ API gateway │ ── │   service     │
└─────┘    └──────────┘    └─────────────┘    └───────────────┘
   ▲          rate-limit         rate-limit         rate-limit
   │
self-throttling (client-side budget)
```

Limit at every tier — defense in depth. Edge for DDoS; gateway for per-API quotas; service for per-tenant fairness.

### Distributed Rate Limiting

When you have N gateway instances, each can't track its own count or you'll allow `N × limit` requests.

Patterns:

**1. Centralized store** (Redis):

```
INCR rate:user:42:60s
EXPIRE rate:user:42:60s 60     (set TTL once)
return value <= limit ? allow : reject
```

Atomic via `INCR`. ~100µs per check. Best for medium scale.

**2. Lua script for token bucket in Redis**:

Atomic refill + consume in one round-trip:

```lua
local tokens = tonumber(redis.call('GET', KEYS[1]) or capacity)
local last   = tonumber(redis.call('GET', KEYS[2]) or 0)
local now    = tonumber(ARGV[1])
tokens = math.min(capacity, tokens + (now - last) * rate)
if tokens >= 1 then
    tokens = tokens - 1
    redis.call('SET', KEYS[1], tokens)
    redis.call('SET', KEYS[2], now)
    return 1
else
    return 0
end
```

**3. Local approximation** (gossip / probabilistic):

Each instance tracks its own counter; periodically share. Used by Stripe, others for very high QPS. Accepts small overshoot for performance.

**4. Sticky routing**:

Hash client IP → always same gateway → local counter sufficient. Doesn't survive instance loss; small overshoot.

### Quotas vs Rate Limits

- **Rate limit**: requests per short window (e.g., 100/min).
- **Quota**: requests per long window (e.g., 10k/day).

Implement both. Quota usually backed by relational DB; rate limit by Redis.

### Response Headers

Tell clients what's happening:

```
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 47
X-RateLimit-Reset: 1700000060

# On rejection:
HTTP/1.1 429 Too Many Requests
Retry-After: 13
```

`Retry-After` lets well-behaved clients back off without retry loops.

### Tiered Limits

Differentiate by plan:

```
free:    10 req/min
pro:     100 req/min
enterprise: 1000 req/min
```

Implementation: limit lookup by `tenant_id` → tier → policy.

### Common Mistakes

| Mistake | Fix |
|---------|-----|
| Limiting by `Remote-Addr` behind a CDN | Use `X-Forwarded-For` correctly, ideally with signed forwarded headers |
| Counting per second only (allows 1-sec spikes) | Pair short + long windows |
| Letting one customer fill the table | Per-tenant quotas |
| No 429 communication | Always return `Retry-After` |
| Blocking abuse with rate limit alone | Layer with WAF, bot detection, CAPTCHAs |

> [!NOTE]
> Rate limiting is part fairness, part protection, part business policy. Don't conflate them. Fairness = per-user; protection = global QPS; policy = tier-based quotas. Build each axis independently.

### Interview Follow-ups

- *"How would you build a global rate limiter for a multi-region service?"* — Per-region local + sync to global counter periodically; allow small overshoot per region.
- *"How do you rate-limit by something other than IP?"* — Composite keys: user_id, API key, JWT claim, fingerprint. Pre-hash to avoid PII in cache.
- *"What if Redis goes down?"* — Fail-open (allow all) for non-abusive scenarios; fail-closed (deny) for security-critical APIs. Pick per endpoint.
