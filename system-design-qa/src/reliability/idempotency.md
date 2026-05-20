# Q: Idempotency — what it is, why it's mandatory, how to enforce it.

**Answer:**

An operation is **idempotent** if performing it multiple times has the same effect as once. In a world of retries, networks that lie, and at-least-once delivery, idempotency isn't optional — it's the only way to guarantee correctness.

### Why It Matters

Networks fail. Clients retry. Brokers redeliver. Without idempotency:

- Double-charges.
- Duplicate orders.
- Counters off by 5×.
- Email sent twice.
- Records inserted N times.

Idempotency turns at-least-once delivery into exactly-once **effect**.

### Naturally Idempotent Operations

- `SET balance = 100`: same value → same state.
- `DELETE WHERE id = 5`: gone is gone.
- `PUT /resource/123` (replacement).

### Naturally NON-Idempotent

- `INCREMENT count`: each call adds.
- `INSERT new row`: each call adds a row.
- `SEND email`: each call sends.
- `POST /charges` (creates): each call creates.

These need explicit dedup.

### Patterns to Make Operations Idempotent

#### 1. Idempotency Keys

Client sends a unique key per logical operation:

```http
POST /charges
Idempotency-Key: 8a4b8c3e-...
{ "amount": 100 }
```

Server stores result keyed by this. Replay returns the cached result.

```python
def charge(req, idempotency_key):
    cached = store.get(idempotency_key)
    if cached:
        return cached.response

    result = do_actual_charge(req)
    store.put(idempotency_key, result, ttl="24h")
    return result
```

The key is client-generated, scoped to (account, key). Same key → same result.

See [Idempotent REST APIs](../../java-qa/src/spring/idempotency.md).

#### 2. Unique Constraints in DB

```sql
CREATE TABLE orders (
    id UUID PRIMARY KEY,
    client_order_id UUID UNIQUE,    -- client-provided
    ...
);

INSERT INTO orders ... ON CONFLICT (client_order_id) DO NOTHING;
```

Duplicate insert → no-op via constraint.

#### 3. Conditional Updates

```sql
UPDATE accounts SET balance = balance - 100
WHERE id = ? AND balance >= 100 AND last_tx_id <> ?
```

Use a `last_tx_id` or version field; idempotent because second call doesn't match.

Or simpler with optimistic concurrency:

```sql
UPDATE row SET ..., version = version + 1
WHERE id = ? AND version = ?
```

#### 4. Event Sourcing / Inbox

Consumer dedups by event ID:

```sql
INSERT INTO inbox (event_id) VALUES (?) ON CONFLICT DO NOTHING;
-- if conflict: ignore
-- else: apply
```

Pairs with at-least-once delivery to give exactly-once effect.

### Idempotency Storage

Where to keep the key → result mapping:

| Store | Trade |
|-------|-------|
| Same DB as data (transactional) | Strongest correctness; same TX |
| Redis (TTL'd) | Fast; loses on Redis crash |
| Dedicated idempotency service | Reusable across services; another dep |

For payments: same DB as the financial data is the only correct answer.

### TTL on Idempotency Records

Records grow forever otherwise. Stripe's choice: 24 hours.

After TTL, a new replay with the same key is treated as a new request. Acceptable because retries usually happen within minutes.

For longer retention (audit), keep a compact log separately.

### HTTP Methods and Idempotency

Per spec:
- GET, PUT, DELETE: idempotent.
- POST, PATCH: not.

In practice, POST is the action that creates side effects — and the one that absolutely needs `Idempotency-Key`.

### Async Pipelines

Same principle in event-driven:

```
producer → at-least-once delivery → consumer
consumer dedups by event_id
```

Without this, every retry = duplicate effect.

### Exactly-Once vs Idempotent

"Exactly-once delivery" = impossible over an unreliable network (Two Generals Problem).

"Exactly-once **effect**" = achievable: at-least-once delivery + idempotent consumer.

The difference is critical in interviews. Don't say "exactly-once delivery" with a straight face.

### Common Mistakes

| Mistake | Reality |
|---------|---------|
| Trusting client to never retry | Networks ensure they will |
| Idempotency by hashing payload | Two unrelated requests can collide; use explicit key |
| Storing idempotency record in cache only | Cache eviction → duplicate processing |
| Idempotency record outside the business transaction | Race: business write succeeds, idempotency record fails → next retry duplicates |
| No TTL | Storage grows forever |

### Validation Pattern

Idempotency record + atomic business write:

```sql
BEGIN;
INSERT INTO idempotency_records (key, status='in_progress', client_id=?)
       VALUES (?, 'in_progress', ?)
ON CONFLICT (key) DO NOTHING;
-- if conflict: return existing record's response

INSERT INTO orders (...);

UPDATE idempotency_records SET status='completed', response=? WHERE key=?;
COMMIT;
```

Returns same response on retry; never creates two orders.

> [!NOTE]
> Every external write endpoint should accept `Idempotency-Key`. Every async consumer should dedup by event ID. Build it into your framework defaults; otherwise it gets forgotten and bites later.

### Interview Follow-ups

- *"What if two distinct operations happen to use the same key?"* — Scope keys per (tenant, client, request). Or accept "first wins" semantics; document it.
- *"How do you handle slow first-call + concurrent retry?"* — Insert idempotency record at "in_progress" state with TTL; second call sees in_progress → return 409 or wait.
- *"What's the difference between idempotency and 2PC?"* — 2PC tries to make multiple steps atomic. Idempotency tries to make one step safe-to-replay. The latter is much cheaper.
