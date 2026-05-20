# Q: REST vs gRPC vs GraphQL — picking an API style.

**Answer:**

The three dominate modern API design. They optimize for different things: REST for **simplicity + caching**, gRPC for **performance + strict contracts**, GraphQL for **flexible client queries**. Most companies use more than one.

### REST

HTTP verbs map to operations on resources, identified by URLs. JSON payload by convention.

```
POST   /v1/orders                 # create
GET    /v1/orders/123             # read one
GET    /v1/orders?status=open     # list with filters
PATCH  /v1/orders/123             # partial update
DELETE /v1/orders/123             # delete
```

Strengths:
- Universal tooling (curl, browsers, CDN).
- Cacheable via standard HTTP (`Cache-Control`, `ETag`).
- Easy to inspect/debug.
- Versionable by URL or header.

Weaknesses:
- No native streaming (long-polling, chunked, or SSE workarounds).
- Type-loose; relies on OpenAPI/JSON Schema for contracts.
- Multiple round-trips for related data (over-fetching / under-fetching).

### gRPC

Defines services in `.proto` files, generates typed clients/servers in any language. Uses HTTP/2 with binary Protobuf payloads.

```protobuf
service OrderService {
  rpc GetOrder(GetOrderRequest) returns (Order);
  rpc ListOrders(ListOrdersRequest) returns (ListOrdersResponse);
  rpc WatchOrders(WatchRequest) returns (stream OrderEvent);   // server stream
  rpc UploadOrders(stream Order) returns (Summary);            // client stream
  rpc Chat(stream Msg) returns (stream Msg);                   // bidi
}
```

Strengths:
- Strong typing, generated code.
- Binary format → smaller, faster than JSON.
- Native streaming (server, client, bidirectional).
- HTTP/2 multiplexing — many calls one connection.
- First-class deadlines, cancellation, metadata.

Weaknesses:
- Browser support requires gRPC-Web proxy (Envoy or in-process).
- Less universal tooling (specialized debuggers needed).
- Schema changes require `.proto` distribution to consumers.

Best for: service-to-service internal APIs, mobile apps with native clients, high-throughput RPC.

### GraphQL

A query language. Clients send a single query asking for exactly the fields they need across multiple resources.

```graphql
query {
  order(id: 123) {
    id
    total
    customer {
      name
      email
    }
    items {
      product { name price }
      quantity
    }
  }
}
```

Strengths:
- Single request fetches a deep object graph.
- Client picks fields — no over-fetching.
- Self-documenting via introspection.
- Strong typing via SDL.

Weaknesses:
- Caching is harder (no URL-keyed cache; needs persisted queries + custom cache layer).
- N+1 query risk on the server (mitigated by DataLoader-pattern batching).
- Authorization is per-field (more surface area).
- "Resolver explosion" — many code paths to maintain.

Best for: aggregating data from many services for a single UI; mobile apps wanting minimal payloads; partner APIs where flexibility matters.

### Comparison Table

| Aspect | REST | gRPC | GraphQL |
|--------|------|------|---------|
| Transport | HTTP/1.1 or 2 | HTTP/2 (required) | HTTP/1.1 or 2 |
| Format | JSON (typical) | Protobuf binary | JSON over HTTP |
| Streaming | SSE / chunked workaround | Native server/client/bidi | Subscriptions (over WebSocket usually) |
| Contract | OpenAPI (optional) | .proto (required) | SDL (required) |
| Code-gen | OpenAPI generators | First-class | Apollo/Relay codegen |
| Caching | Browser/CDN native | Hard | Hard (persisted queries) |
| Debugging | curl, browser dev tools | grpcurl, BloomRPC | GraphiQL, IDE plugins |
| Versioning | URL/header | Backward-compatible .proto rules | Field deprecation; never break |

### Versioning

**REST**: `/v1/orders`, `/v2/orders` — old and new run side by side.

**gRPC**: Protobuf evolution rules. Add fields with new tag numbers; never reuse or change types. Old clients ignore unknown fields.

**GraphQL**: never version; deprecate fields with `@deprecated`, add new ones. The schema evolves like an org chart.

### Error Models

**REST**: HTTP status codes + JSON body.
- 4xx client error, 5xx server error.
- RFC 7807 "Problem Details" for structured errors.

**gRPC**: status codes (OK, INVALID_ARGUMENT, NOT_FOUND, ABORTED, ...).
- Rich error model via google.rpc.Status with details.

**GraphQL**: always 200 OK with `errors` array in body. Has to define partial-success.

### Streaming Comparison

| Need | REST | gRPC | GraphQL |
|------|------|------|---------|
| One-shot request/response | Native | Native | Native |
| Server pushes events | SSE or polling | server stream | subscription |
| Client streams uploads | Chunked transfer | client stream | rare (multipart) |
| Bidirectional chat | WebSocket | bidi stream | subscription |

### When to Pick Each

```
Internal service-to-service          → gRPC
Public REST API for many consumers   → REST + OpenAPI
Mobile app with constrained network  → GraphQL or gRPC (compact payloads)
Browser fetch, hit-and-run           → REST
Real-time bidirectional              → gRPC or WebSocket
Aggregation across many backends     → GraphQL (BFF pattern)
```

### Hybrid Architectures Are Normal

```
Browser ──REST/GraphQL──► Edge / BFF ──gRPC──► Services ──gRPC──► Services
                                                │
                                                ▼
                                              DB / cache
```

- REST or GraphQL on the edge for browser ergonomics.
- gRPC internally for performance and contracts.

### Pagination

REST: `?cursor=...&limit=...` (cursor-based) or `?page=...&size=...` (offset-based).

gRPC: PageToken pattern.

GraphQL: Relay Cursor Connection spec — connections, edges, page info.

Cursor-based scales; offset-based breaks past a few thousand rows.

### Idempotency

POST is non-idempotent by default. All styles handle this the same way:

- Client sends `Idempotency-Key` header / metadata.
- Server stores result keyed by it; replays on retry.

See [Idempotent REST APIs](../../java-qa/src/spring/idempotency.md).

### Common Mistakes

| Mistake | Fix |
|---------|-----|
| Using GET for actions with side effects | Verb hygiene; CDN may "pre-fetch" |
| REST + no OpenAPI spec | Drift between docs and reality; generate from spec |
| gRPC over the open internet without TLS | Always use TLS; gRPC was designed assuming it |
| GraphQL queries 5 levels deep on every page | Set depth/complexity limits; reject expensive queries |
| Switching everything to GraphQL "for flexibility" | Old REST endpoints + caching are still right for hot read paths |

> [!NOTE]
> Pick by who consumes your API and what they need. Internal microservices = gRPC. Diverse external consumers = REST. UI aggregating lots of data = GraphQL. There is no general winner.

### Interview Follow-ups

- *"How do you do auth in gRPC?"* — Metadata headers (often a JWT). Interceptors verify per-call. mTLS for service-to-service.
- *"What's HATEOAS and do you use it?"* — Hypermedia in REST responses (links to next actions). Theoretically elegant; rarely worth the complexity in practice.
- *"How does GraphQL handle file uploads?"* — Multipart spec; usually offload to a separate REST endpoint that returns a URL.
