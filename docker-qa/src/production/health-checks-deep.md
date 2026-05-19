# Q: How do you design good container health checks (liveness vs readiness vs startup)?

**Answer:**

Health checks decide whether a container gets traffic, gets restarted, or gets killed. Conflating the three kinds — **startup**, **readiness**, **liveness** — is the most common cause of restart loops, traffic served before warmup, and outages masked as transient flaps.

### The Three Kinds (Kubernetes Terminology)

| Kind | Question | What happens on fail |
|------|----------|---------------------|
| **startup** | Has the app finished booting? | Wait longer; only after success do liveness/readiness start |
| **readiness** | Should this pod receive traffic right now? | Remove from Service endpoints; pod stays alive |
| **liveness** | Is the process broken beyond recovery? | Kill the pod; restart |

Docker (standalone / Compose) only has one: `HEALTHCHECK`. Cluster orchestrators (K8s, ECS) have all three.

### Docker `HEALTHCHECK`

```dockerfile
HEALTHCHECK --interval=30s --timeout=3s --start-period=20s --retries=3 \
  CMD curl -fsS http://localhost:8080/health || exit 1
```

Flags:
- `--interval`: between checks (default 30s).
- `--timeout`: max time per check (default 30s).
- `--start-period`: grace period for startup; failures here don't count (default 0s).
- `--retries`: consecutive failures before `unhealthy` (default 3).

Compose binds Docker's healthcheck to `depends_on`:

```yaml
services:
  db:
    image: postgres
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      retries: 10
  app:
    image: myapp
    depends_on:
      db:
        condition: service_healthy
```

`service_healthy` waits for `db` to be `healthy` before starting `app`. Eliminates the "wait-for-it.sh" pattern.

### Why Three Probes in K8s

Spring Boot example:

- Startup: app takes 30s to load DB schema, warm caches.
- Ready: must be able to reach DB and Kafka.
- Live: process must not be deadlocked.

Without a startup probe, you'd set liveness timeout high enough to cover boot (60s) — but then a *real* deadlock takes 60s to detect. With a startup probe, you split: boot waits at the startup probe, then liveness goes tight (5s timeout).

### Kubernetes Example

```yaml
containers:
- name: app
  image: myapp:1.2
  startupProbe:
    httpGet: { path: /actuator/health/liveness, port: 8080 }
    periodSeconds: 5
    failureThreshold: 30        # 30 × 5s = 150s total grace
  readinessProbe:
    httpGet: { path: /actuator/health/readiness, port: 8080 }
    periodSeconds: 5
    failureThreshold: 3
  livenessProbe:
    httpGet: { path: /actuator/health/liveness, port: 8080 }
    periodSeconds: 10
    failureThreshold: 3
    timeoutSeconds: 2
```

### What Each Endpoint Should Check

**Liveness** — minimal. Only fails if the process is truly broken.
- Process responds (the act of replying is the test).
- Not stuck in an infinite loop / deadlock.
- **Do NOT** check downstreams. If the DB is down, killing every replica makes it worse.

**Readiness** — full check.
- Can talk to DB, queue, cache.
- Migrations applied.
- Warmup done.
- During shutdown, return failure *before* the app starts rejecting connections (gives load balancer time to drain).

**Startup** — same as readiness, but longer grace window.

### Spring Boot Built-Ins

```yaml
management:
  endpoint:
    health:
      probes:
        enabled: true
```

Endpoints exposed:
- `GET /actuator/health/liveness`
- `GET /actuator/health/readiness`

`ApplicationAvailability` API lets your code emit events that change the state:

```java
@Component
class Listener {
    @EventListener
    void onDbDown(DbDownEvent e) {
        AvailabilityChangeEvent.publish(ctx, ReadinessState.REFUSING_TRAFFIC);
    }
}
```

### Choosing Check Type

| Check command | When |
|--------------|------|
| `curl localhost:port/health` | HTTP services |
| `pg_isready` / `mongo --eval ping` | Datastores with built-in probes |
| `nc -z localhost 6379` | TCP-only services |
| Custom script | Composite checks |

For non-HTTP containers, a TCP probe is fine — connection accept = container alive.

### Graceful Shutdown + Readiness

```
SIGTERM received
   │
   ▼ readiness probe should start failing  ← critical
   │
   │  ↓ load balancer removes pod from endpoints
   │  ↓ in-flight requests drain
   │
   ▼ stop accepting new connections
   ▼ flush metrics
   ▼ exit(0)
```

Spring Boot:

```yaml
server:
  shutdown: graceful
spring:
  lifecycle:
    timeout-per-shutdown-phase: 30s
```

Combined with `terminationGracePeriodSeconds: 45` in K8s.

### Anti-Patterns

| Pattern | Why bad |
|---------|---------|
| Liveness probe checks DB | DB outage = restart loop of all pods → makes recovery slower |
| Same endpoint for liveness + readiness | Liveness false-positives on transient downstream issues |
| Startup probe → liveness with no startup probe → tight liveness | Pod killed before it finishes booting |
| `failureThreshold: 1` on liveness | One transient blip = restart. Use ≥ 3 |
| Health endpoint requires auth | Probes can't authenticate; expose unauthenticated or use exec probe |
| Heavy work in health handler (joins, full diagnostics) | Slow handler → probe timeout → false failure |

### Probes vs Application Layer

In a service mesh (Istio, Linkerd), readiness controls routing. In bare K8s, it controls Service endpoint membership. Either way, *readiness affects traffic; liveness affects life*.

### Compose `depends_on` Conditions

```yaml
depends_on:
  db:
    condition: service_healthy        # waits for healthcheck pass
  migrator:
    condition: service_completed_successfully   # waits for one-shot exit 0
```

The second pattern is excellent for DB migrations: a one-shot `migrator` service runs `flyway migrate`, exits 0, then app starts.

### Diagnostics

```bash
# Check container health
docker ps --format "table {{.Names}}\t{{.Status}}"
# myapp   Up 5 minutes (healthy)

# Last few health probe results
docker inspect --format='{{json .State.Health}}' myapp | jq

# K8s:
kubectl describe pod myapp           # Events show probe failures
kubectl logs myapp                   # See what the app reports
```

> [!NOTE]
> A good rule: liveness should be the *narrowest* probe (just "am I responding?"). Readiness should be the *fullest* (can I actually serve a request end-to-end?). The asymmetry prevents restart loops while still gating traffic correctly.

### Interview Follow-ups

- *"What if a backend dependency is intermittent?"* — Readiness fails → traffic stops → recovers when it comes back. Liveness never failed → no restart. Exactly the desired behavior.
- *"How do you handle multi-container pods?"* — Each container has its own probes. Pod is `Ready` only when *all* containers are ready.
- *"`exec` vs `httpGet` vs `tcpSocket` probes?"* — `exec` forks a process inside the container (expensive, but useful for CLI-only checks). `httpGet` is cheapest and most informative. `tcpSocket` is a minimalist liveness check.
