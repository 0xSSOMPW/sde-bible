# Q: API Gateway and BFF (Backend for Frontend) — what role do they play?

**Answer:**

In a microservices world, clients can't talk to 100 backend services directly. The **API gateway** is the front door: one stable endpoint that routes, authenticates, rate-limits, and aggregates. The **BFF** is a specialization: one gateway per client type (web, iOS, Android).

### API Gateway

```
                       ┌──────────────┐
client ───────────────►│ API Gateway  │
                       └──────┬───────┘
                              │
        ┌─────────────┬───────┼─────────┬───────────┐
        ▼             ▼       ▼         ▼           ▼
     Orders      Inventory  Search    Payments   Users
     Service     Service    Service   Service    Service
```

Responsibilities:
- **Routing**: URL/path/host → service.
- **AuthN / AuthZ**: verify JWT; map to user context.
- **Rate limiting**: per-IP, per-API-key, per-tenant.
- **Request shaping**: header injection, request transformation.
- **Response shaping**: aggregate, filter.
- **Caching**: HTTP-cache headers for read-mostly endpoints.
- **Observability**: per-route metrics, traces.
- **Circuit breaking / retries** (optional).

Popular implementations: AWS API Gateway, Kong, Envoy, Tyk, Apigee, custom-built.

### BFF Pattern

Different clients need different shapes of data. Rather than one gateway serving all, **one BFF per client**:

```
                    ┌──── Web BFF      ─► Services
                    │
client (web) ───────┘

                    ┌──── Mobile BFF   ─► Services
                    │
client (iOS) ───────┘

                    ┌──── Partner BFF  ─► Services
                    │
client (partner)────┘
```

Each BFF:
- Owned by the client team (closer to UI requirements).
- Optimized for that client (smaller payloads on mobile; richer on web; restricted on partner).
- Aggregates backend services into one client-friendly response.

GraphQL is often the right tool for the BFF layer — clients pick what they need.

### Why Not "Just Have Services Call Each Other"

Pre-gateway client-direct era problems:
- Clients hardcoded N service URLs.
- Each service handled auth differently.
- Cross-cutting concerns (rate limit, retries) duplicated.
- Browser CORS hell.

Gateway centralizes that. Worth the extra hop.

### Anti-Patterns

| Anti-pattern | What goes wrong |
|--------------|----------------|
| Business logic in gateway | Gateway becomes monolith over time |
| Stateful gateway | Doesn't scale; defeats purpose |
| Single gateway for browser + mobile + partners | Different needs collide; use BFF per client |
| Gateway that retries everything | Hidden duplication; idempotency required |
| Synchronous fanout across many services per request | Latency multiplied; use aggregation patterns |

### Aggregation

```
GET /v1/pages/order/123
   ├── parallel call: orders.get(123)
   ├── parallel call: payments.byOrder(123)
   ├── parallel call: shipping.byOrder(123)
   ├── parallel call: users.get(order.user_id)
   └── compose response
```

Gateway/BFF orchestrates; one client request → N parallel service calls → one merged response.

### Auth at the Gateway

Standard pattern:

```
client → gateway
   gateway: verify JWT signature + expiry
   gateway: extract claims → set X-User-Id header
   gateway: forward to service

service: trust X-User-Id from gateway (internal network)
```

Service shouldn't re-verify (slow); but gateway must be authoritative.

mTLS between gateway and services prevents spoofing internal headers.

### Rate Limiting

Multi-level:
- Per-IP at edge.
- Per-API-key / per-token at gateway.
- Per-user / per-tenant at gateway or BFF.

See [Rate Limiting Algorithms](../patterns/rate-limiting.md).

### Versioning

Gateway-level versioning lets you migrate services behind it:

```
/v1/orders → orders-service-v1
/v2/orders → orders-service-v2
```

Or header-based:

```
Accept-Version: 2
```

### Gateway as Performance Layer

- **Compression** (gzip/br).
- **HTTP/2 multiplexing** to upstream.
- **TLS termination** centralized.
- **Edge caching** for read-mostly responses.
- **Connection pooling** to backends.

### Failure Modes

| Failure | Handling |
|---------|---------|
| One service down | Circuit-break; return partial response if possible |
| Gateway pod dies | LB removes; remaining pods absorb |
| Auth service slow | Cache JWT validations briefly; fail fast |
| Single point of failure | Multi-AZ deployment; minimum 3 instances |

### Common Mistakes

| Mistake | Fix |
|---------|-----|
| Gateway becomes monolith of "common code" | Keep it thin; cross-cutting only |
| Service mesh + gateway with overlapping responsibilities | Define clear roles |
| BFF that grows beyond its team | Refactor into smaller services |
| Direct DB access from gateway | Never; gateway is stateless |
| No versioning strategy | Painful migrations |

> [!NOTE]
> The gateway is your platform's contract with the outside world. Keep its responsibilities narrow (routing, auth, rate-limit, basic shape); push everything domain-specific into services or BFFs.

### Interview Follow-ups

- *"Where do you put logging / tracing?"* — Gateway emits per-request entries; trace context propagated via headers (W3C `traceparent`) to all downstream calls.
- *"How is API Gateway different from a service mesh?"* — Gateway = north-south traffic (client → cluster). Mesh = east-west (service ↔ service). Different layers; often coexist.
- *"How do you avoid the gateway becoming a SPOF?"* — Multi-region, multi-AZ; managed (cloud) when possible; client retries with backoff.
