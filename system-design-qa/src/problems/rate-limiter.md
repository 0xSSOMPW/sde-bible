# Q: Design a distributed rate limiter.

**Answer:**

Standard interview problem combining algorithms (token bucket etc.) with distributed-system design (where state lives, how to scale).

### Requirements

**Functional**:
- Reject requests above the threshold per identifier (user, IP, API key).
- Multiple tiers (free 10/min, pro 100/min).
- Support short windows (per second) and long windows (per day).
- Return clear error with `Retry-After`.

**Non-functional**:
- Latency overhead < 5 ms p99 (it's in every request path).
- Survive a Redis outage gracefully.
- Globally consistent enough across many gateway pods (some overshoot OK).
- Scale to 1M+ QPS limiter checks.

### High-Level Design

```
                ┌──────────────┐
client ───────► │  API Gateway │ ────► origin service
                └──────┬───────┘
                       │ check limit
                       ▼
                ┌──────────────┐
                │ Limiter logic │ (in process)
                └──────┬───────┘
                       │ atomic op
                       ▼
                ┌──────────────┐
                │ Redis Cluster │
                └──────────────┘
```

Limiter logic runs in-process at the gateway; state is in Redis (shared across pods).

### Algorithm Choice

For most use cases: **token bucket** (allows bursts) or **sliding window counter** (smoother). See [Rate Limiting Algorithms](../patterns/rate-limiting.md).

Default pick: **sliding window counter** for fairness + O(1) memory.

### Atomic Counter in Redis

```
key = "rl:user:{user_id}:{minute_bucket}"

INCR key
EXPIRE key 120                  ← set on first INCR to avoid drift

if value > limit:
    reject
```

Atomic via `INCR`. Two ops; round-trip ~0.5–1 ms in DC.

Bonus: use `INCR` + `EXPIRE` in one Redis pipeline.

### Token Bucket via Lua

Atomic refill + consume in one script:

```lua
local key      = KEYS[1]
local capacity = tonumber(ARGV[1])
local rate     = tonumber(ARGV[2])
local now      = tonumber(ARGV[3])

local data = redis.call('HMGET', key, 'tokens', 'ts')
local tokens = tonumber(data[1]) or capacity
local ts     = tonumber(data[2]) or now

tokens = math.min(capacity, tokens + (now - ts) * rate)

if tokens >= 1 then
    tokens = tokens - 1
    redis.call('HMSET', key, 'tokens', tokens, 'ts', now)
    redis.call('EXPIRE', key, 60)
    return 1     -- allowed
else
    return 0     -- rejected
end
```

One round-trip, atomic, deterministic.

### Sliding Window Counter

```
prev_window = floor((now - window_size) / window_size)
curr_window = floor(now / window_size)

prev_count = GET "rl:user:42:" + prev_window
curr_count = GET "rl:user:42:" + curr_window

elapsed_in_curr = (now % window_size) / window_size
weighted = prev_count * (1 - elapsed_in_curr) + curr_count

if weighted >= limit:
    reject
else:
    INCR curr_window
    EXPIRE 2 * window_size
```

Two reads + one write; close to perfect smoothing.

### Scale to 1M+ QPS

Single Redis caps around 100–200k ops/sec/instance. Strategies:

**1. Redis Cluster.**
Shard keys by user_id hash. Limiter calls go to the right shard.

**2. Local + Redis hybrid.**
- Each gateway pod tracks local count.
- Periodically (every 100 ms) reconcile with Redis.
- Accepts small overshoot (matters for advisory limits, not security).

**3. Per-tier strategies.**
- Per-IP: cheap; LRU cache + Redis.
- Per-API-key: more important, full Redis precision.
- Per-tenant quota (daily): Postgres or a quota service.

### Handle Redis Outage

Two failure modes to choose from:

| Mode | Behavior on Redis down |
|------|------------------------|
| Fail-open | Allow all requests (no limiting) |
| Fail-closed | Reject all requests (deny by default) |

For DDoS protection: fail-open is dangerous; abuser gets a window of unbounded access. Mitigate with local circuit breaker + secondary cluster.

For free-tier quotas: fail-open is usually fine (revenue not at risk).

Pick per use case. Document the decision.

### Distributed Limiter Patterns

**1. Centralized counter** (Redis): what we described.

**2. Token-bucket-on-leader-then-broadcast**:
Pick one node as leader; it owns the bucket. Other nodes ask. Adds latency; rarely worth it.

**3. Random tax / probabilistic limiting**:
At high QPS, sampling-based limiting is cheaper than per-request counting.

**4. Local + sync** (Stripe pattern):
Each pod tracks local bucket; periodically syncs with global pool. Accepts brief overshoot for huge throughput.

### Rate-Limit Headers

```
X-RateLimit-Limit:       100
X-RateLimit-Remaining:   72
X-RateLimit-Reset:       1700000123          (unix time of next window)

On 429:
HTTP/1.1 429 Too Many Requests
Retry-After: 13                              (seconds)
```

Good clients honor `Retry-After`. Bad clients don't; they get a circuit breaker.

### Tier Lookup

```
GET /users/{id}
  → user.plan = "pro"

policies = {
  free: { rpm: 10,  daily: 1000 },
  pro:  { rpm: 100, daily: 100000 }
}

policy = policies[user.plan]
```

Cache the policy. Plan changes can invalidate via a short TTL or pub/sub.

### Multi-Region

Each region runs its own limiter. Two questions:

**1. Per-region quota or global?**

```
Per-region: each region enforces 100 rps independently → 100 × N regions = N × allowed
Global:     central counter (still Redis) → cross-region latency added
```

For most APIs, per-region is fine. For security-critical (auth/login), global with eventual penalty.

**2. Global cap, regional enforcement**:
- Allocate each region 80% of its share to start.
- Bidirectional gossip to redistribute unused capacity.
- Stripe / Cloudflare patterns.

### Edge Rate Limiting

Closer to the user = better DDoS mitigation.

- **L4** (Cloudflare, AWS Shield): connection-level limit, very fast.
- **L7 at CDN edge**: per-IP/per-URL.
- **At gateway**: per-API-key, per-tenant.
- **At service**: per-endpoint, fine-grained.

Layered defense. Each tier protects the next.

### Failure Modes & Mitigations

| Failure | Mitigation |
|---------|-----------|
| Redis slow → request latency spike | Aggressive timeout (5 ms); fail-open with local fallback |
| Hot key (one user with massive traffic) | Shard their counter (`bucket = user_id + random_n`); aggregate |
| Limiter itself becomes target | Front with CDN / WAF |
| Counter never expires (bug) | TTL set on every INCR; sanity job clears stale |

### Observability

Metrics:
- `rate_limit_allowed_total`, `rate_limit_rejected_total` per route + tier.
- p99 limiter latency.
- Redis hit ratio, latency.
- Top rejected sources (IP, API key) for abuse investigation.

### Common Mistakes

| Mistake | Fix |
|---------|-----|
| INCR without EXPIRE | Counter persists forever |
| Per-pod limits in front of global LB | N pods × limit = wrong total |
| 429 with no `Retry-After` | Clients hammer; cascading retries |
| Rate-limit by IP behind a proxy | Use `X-Forwarded-For`, with signature/trust |
| Same limit for all routes | Heavy endpoints need tighter limits than light ones |

> [!NOTE]
> Build the limiter as a library, not a separate service. Network hop per request is too expensive. Keep state in Redis but logic in-process.

### Interview Follow-ups

- *"Could you build this without Redis?"* — Distributed token bucket in-process with gossip (Stripe-style). More complex; appropriate at massive scale.
- *"How does it interact with caching?"* — Limiter sits before cache; counts every request including cache hits. Or count only origin-bound requests if your use case allows.
- *"How would you debug 'why is my user getting throttled?'"* — Per-request log line with bucket key + count + limit; tail it filtered by user_id.
