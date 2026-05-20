# Q: Load balancers — L4 vs L7, algorithms, health checks, gotchas.

**Answer:**

A load balancer (LB) distributes incoming requests across multiple backend instances. The decision tree starts at **OSI layer**: L4 (transport) sees IP + port; L7 (application) sees HTTP headers, paths, cookies.

### L4 vs L7

| Aspect | L4 (TCP/UDP) | L7 (HTTP/HTTPS) |
|--------|-------------|-----------------|
| Inspects | IP, port, protocol | URL, headers, cookies, payload |
| Latency overhead | µs | ms (small) |
| TLS termination | Pass-through or terminate | Always terminates |
| Routing rules | Backend pool by VIP/port | Path, host, header-based |
| Cost | Cheap, line-rate | More CPU |
| Examples | AWS NLB, HAProxy (mode tcp), IPVS, MetalLB | AWS ALB, Envoy, Nginx, Traefik, HAProxy (mode http) |

L4 for raw throughput, non-HTTP protocols, deep packet workloads.
L7 for everything HTTP — that's where the smart routing pays off.

### Common Algorithms

| Algorithm | When |
|-----------|------|
| Round Robin | Default; homogeneous backends |
| Weighted Round Robin | Heterogeneous (some larger boxes) |
| Least Connections | Sticky workloads (long-lived TCP) |
| Least Response Time | Auto-balance based on observed latency |
| Random | Surprisingly effective; no state |
| Random Two Choices ("Power of Two") | Pick 2 random, send to the less loaded — near-optimal, low overhead |
| IP Hash / Consistent Hash | Sticky sessions, cache locality |
| Resource-based | Drain by CPU/memory headroom (advanced) |

For most web services: **Power of Two Choices** is the modern default. Beats round-robin and least-connections at scale with O(1) memory.

### Health Checks

LB must know which backends are alive.

**Active**: probe each backend on an interval.
- HTTP `GET /health` every 2–5s.
- Timeout: ≤ 1s typical.
- Mark unhealthy after N consecutive failures (3–5).
- Mark healthy after M consecutive successes (2–3) to avoid flapping.

**Passive**: observe live traffic.
- Mark unhealthy if outbound error rate spikes.
- Cheaper but slower to react.

Best practice: both — active for fast detection, passive for nuance.

The probe endpoint should reflect what readiness means for traffic. See [Health Checks Deep Dive](../../docker-qa/src/production/health-checks-deep.md).

### Sticky Sessions

Pin one client to one backend across requests. Use when:

- The backend has in-memory session state that isn't replicated.
- Cache warmth matters (consistent hashing).

Mechanisms:

- **Source IP affinity** (L4): brittle, breaks behind NAT.
- **Cookie-based** (L7): LB sets/reads a `lb-session` cookie.
- **Consistent hash on key** (e.g., user_id in header).

Cost: uneven load when one user spikes. Prefer stateless services + shared cache when possible.

### TLS Termination

Two strategies:

```
A) Terminate at LB (most common)
   client ──TLS──► LB ──HTTP──► backend
   - Cert lives at LB
   - Backend sees plaintext
   - Easy to inspect, route, gzip

B) Passthrough / SNI routing
   client ──TLS──► LB ──TLS──► backend
   - LB only routes by SNI
   - End-to-end encryption
   - No L7 features
```

Re-encrypt option: terminate, decrypt, then open a new TLS connection to backend. Costs CPU but lets the LB inspect while keeping in-DC traffic encrypted.

### Global vs Regional LBs

```
DNS / Anycast (global)
   │
   ▼
GeoDNS / AWS Global Accelerator
   │
   ▼ routes to nearest region
Regional LB (ALB, NLB, Envoy)
   │
   ▼
Backend instances
```

Global routing handled by **DNS** (latency-based, geo-based) or **Anycast IP** (BGP-based). Regional LB then distributes within the region.

### LB as a Failure Domain

The LB is itself a service. Mitigate:

- **Multiple LB instances** behind a VIP / DNS round-robin.
- **Cloud-managed LB**: AWS ALB/NLB are themselves multi-AZ by default.
- **Self-managed**: keepalived + VIP failover, BGP ECMP for active-active.

A single Nginx box is *not* a load balancer for production.

### Service Mesh as Per-Pod LB

In K8s + Istio/Linkerd, each pod has a sidecar that handles client-side load balancing. The "global" LB becomes the cluster ingress; everything inside is mesh-routed.

Benefits:
- Connection pooling per pod.
- Per-call retries, timeouts, mTLS without app code.
- Power-of-two-choices natively.

Cost:
- Sidecar resource overhead.
- More moving parts to debug.

### Common Gotchas

| Gotcha | Fix |
|--------|-----|
| Slow start: new backend gets full traffic instantly | Configure slow-start ramp on LB |
| Health-check passes but the app is "warming up" | Use startup probe (K8s) or distinct `/ready` endpoint |
| Sticky session sends all whales to one node | Add hot-shard splitting; don't rely on stickiness for fairness |
| Single LB instance | Multi-AZ HA; managed LB or active-passive VIP |
| Health check too aggressive | Causes flap; use higher thresholds + windows |
| Long-lived connections imbalance | Use Least Connections + connection age limits |

### Connection Draining

When pulling a backend out (deploy, scale-in), don't kill in-flight connections.

```
1. Mark backend "draining" — LB stops sending new connections
2. Wait for existing to finish (or hit timeout, e.g. 60s)
3. Kill backend
```

K8s: combine with `preStop` hook + `terminationGracePeriodSeconds`.

### LB Architecture Reference

```
                  Internet
                     │
                     ▼
              ┌──────────────┐
              │ Anycast / DNS │  (global)
              └──────┬───────┘
                     │
        ┌────────────┼────────────┐
        ▼            ▼            ▼
    us-east-1     us-west-2    eu-west-1
        │            │            │
        ▼            ▼            ▼
     ALB/NLB      ALB/NLB      ALB/NLB
        │            │            │
        ▼            ▼            ▼
  K8s ingress  K8s ingress  K8s ingress
        │            │            │
        ▼            ▼            ▼
     pods         pods         pods
```

### Common Mistakes

| Mistake | Reality |
|---------|---------|
| Using L7 features for non-HTTP traffic | L4 with hashing is what you want |
| Forgetting WebSocket support | Need WS-aware LB (ALB, Envoy); some legacy LBs break upgrades |
| HTTP/2 from client but HTTP/1.1 to backend | Possible (most LBs translate) but watch for streaming-RPC mismatches |
| Per-route routing in app code | Move it to L7 LB; cleaner, more reliable |
| Direct LB → DB | LBs are for stateless backends; DBs use their own replicas |

> [!NOTE]
> A surprisingly large portion of "scaling problems" are LB problems: hot backends, bad sticky sessions, slow health checks. Treat the LB as a first-class component to design, not an afterthought.

### Interview Follow-ups

- *"How do you load-balance gRPC?"* — gRPC uses HTTP/2 long-lived connections; round-robin breaks. Use Envoy with per-request L7 LB, or client-side LB.
- *"How do you handle a 'thundering herd' on backend restart?"* — Slow-start, jittered staggering, warmup endpoints.
- *"How do you do canary deploys via LB?"* — Weighted target groups (1% → 10% → 50% → 100%) or header-based routing for a small group of users.
