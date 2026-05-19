# Q: How do you build idempotent REST APIs with the Idempotency-Key pattern?

**Answer:**

A request is **idempotent** if making it twice has the same effect as making it once. For unsafe HTTP methods (POST, PATCH) that's not free — you have to design for it. The standard pattern, adopted by Stripe and now formalized as IETF draft `draft-ietf-httpapi-idempotency-key-header`, uses an `Idempotency-Key` header.

### Why It Matters

Network failures don't tell you whether your write succeeded. The client retries, the server processes a second `POST /payments` → double-charge. The fix can't live in the client (timeouts aren't reliable) or in the network (retries are necessary). It has to live in the server.

```
Client          Server
  │ POST /payments  ─────────►
  │                            charge OK
  │ ◄ ── ── ── X (TCP reset)
  │
  │ retry POST /payments ────►
  │                            charge AGAIN  <-- bug
```

### The Contract

```
POST /payments
Idempotency-Key: 8a4b8c3e-...
Content-Type: application/json

{ "amount": 100, "currency": "USD" }
```

Server promise:
1. If this key was **never seen**, perform the operation, store the response, return it.
2. If this key was **seen with the same request body**, return the **stored** response — do not perform the operation again.
3. If this key was seen with a **different body**, return `422 Unprocessable Entity` (key reuse with conflict).
4. If a request with this key is **in flight**, return `409 Conflict` or wait.

### Data Model

```sql
CREATE TABLE idempotency_records (
    key             VARCHAR(64)  PRIMARY KEY,
    request_hash    VARCHAR(64)  NOT NULL,
    status_code     SMALLINT     NOT NULL,
    response_body   TEXT         NOT NULL,
    created_at      TIMESTAMP    NOT NULL,
    expires_at      TIMESTAMP    NOT NULL
);
```

- `key` is the client-supplied header.
- `request_hash` is SHA-256 over the canonicalized request payload (used to detect (3)).
- `expires_at` lets you GC keys after 24h–7d (per Stripe convention).

### Spring Implementation Sketch

```java
@Component
@RequiredArgsConstructor
public class IdempotencyFilter extends OncePerRequestFilter {

    private final IdempotencyStore store;
    private final ObjectMapper mapper;

    @Override
    protected void doFilterInternal(HttpServletRequest req,
                                    HttpServletResponse resp,
                                    FilterChain chain) throws IOException, ServletException {

        if (!isUnsafeMethod(req)) { chain.doFilter(req, resp); return; }

        String key = req.getHeader("Idempotency-Key");
        if (key == null) { chain.doFilter(req, resp); return; }

        var cached = new ContentCachingRequestWrapper(req);
        byte[] body = cached.getContentAsByteArray();
        String hash = sha256(body);

        Optional<IdempotencyRecord> existing = store.find(key);
        if (existing.isPresent()) {
            var rec = existing.get();
            if (!rec.requestHash().equals(hash)) {
                resp.setStatus(422);
                resp.getWriter().write("{\"error\":\"idempotency key reused with different payload\"}");
                return;
            }
            resp.setStatus(rec.statusCode());
            resp.getWriter().write(rec.responseBody());
            return;
        }

        // First time: capture response, then store
        var respWrapper = new ContentCachingResponseWrapper(resp);
        chain.doFilter(cached, respWrapper);

        if (respWrapper.getStatus() < 500) {
            store.save(new IdempotencyRecord(
                key, hash, respWrapper.getStatus(),
                new String(respWrapper.getContentAsByteArray())));
        }
        respWrapper.copyBodyToResponse();
    }
}
```

### Concurrent Requests with the Same Key

A naïve check-then-write race lets two requests both pass the "not present" check. Two defenses:

**1. Insert-first lock row.**

```sql
INSERT INTO idempotency_records(key, status_code, response_body, ...)
VALUES (?, 0, '', ...)
ON CONFLICT DO NOTHING
RETURNING key;
```

- If you got a row, you own the operation.
- If `INSERT ... DO NOTHING` returned nothing, someone else is processing — return `409` or poll the row.

**2. Distributed lock (Redis SETNX).**

```
SET idempotency:KEY processing NX EX 60
```

Lighter weight but adds another dependency.

### Scope of Idempotency

| Scope | Example |
|-------|---------|
| Per resource | `POST /accounts/{id}/payments` — key valid only for that account |
| Per tenant | Key namespaced by `tenant_id` |
| Global | Rarely sensible |

Always include the **authenticated user/account** in the key namespace, otherwise one user's key collides with another's.

### Response Replay vs Re-execution

Two interpretations:

- **Strict idempotency**: replay the *stored response bytes*, even if the underlying resource has since changed. (Stripe does this.)
- **Effect idempotency**: re-execute the operation, rely on uniqueness constraints to no-op. (Simpler, but the response can differ.)

Strict is what clients expect.

### Common Mistakes

| Mistake | Fix |
|---------|-----|
| Storing response *before* operation commits | Use the same DB transaction for business write + idempotency record |
| Idempotency middleware around `5xx` responses | Don't cache server errors — let client retry |
| Treating GET as needing idempotency | GET is already idempotent by definition |
| No expiry on idempotency records | Table grows forever; set TTL (Stripe: 24h) |
| Hashing raw body bytes including timestamps | Canonicalize JSON first (sort keys, normalize whitespace) |

> [!NOTE]
> Idempotency-Key is for **at-least-once delivery semantics over an at-most-once business operation**. It doesn't replace transactional outbox or saga patterns — it's the request-layer half of those.

### Interview Follow-ups

- *"How is this different from `@Transactional`?"* — `@Transactional` makes DB writes atomic. Idempotency-Key makes the *whole HTTP request* replayable. They're orthogonal; you typically use both.
- *"Why not let clients use a unique business ID instead?"* — They should — for the business object — but you still need an envelope key for the HTTP retry, because the business write might or might not have happened on the first attempt.
- *"Where do you put the idempotency store?"* — Same DB as the business data, so a single transaction covers both. Redis is faster but introduces a two-phase-commit problem.
