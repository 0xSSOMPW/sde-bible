# Q: Saga pattern — choreography vs orchestration. When and how?

**Answer:**

A Saga manages a **multi-step business transaction across services** without distributed 2PC. Each step is a local transaction; failures trigger **compensating actions** to undo earlier steps. Two flavors: **choreography** (event-driven, no coordinator) and **orchestration** (central coordinator drives the flow).

### The Problem

```
Place order = {
  reserve inventory   (service A)
  charge card         (service B)
  ship                (service C)
  notify              (service D)
}
```

A relational transaction across A, B, C, D would need 2PC (XA) — slow, fragile, doesn't work across HTTP. Saga decomposes into 4 local transactions + 4 compensations.

### Choreography (Event-Driven)

Each service publishes events when its step completes; other services subscribe and react.

```
order-service ──OrderPlaced──► inventory-service
                                  │
                                  ▼
inventory-service ──InventoryReserved──► payment-service
                                            │
                                            ▼
payment-service ──PaymentCharged──► shipping-service
                                       │
                                       ▼
                          ShipmentScheduled
```

On failure:

```
payment-service ──PaymentFailed──► inventory-service
                                       │
                                       ▼
                              compensate: release stock
                              publish: InventoryReleased
                                       │
                                       ▼
order-service ──OrderFailed──► notify-service
```

Pros:
- No central component.
- Services loosely coupled.
- Easy to add new participants.

Cons:
- Hard to reason about the full flow.
- Hard to debug (where is my order stuck?).
- Cyclic event dependencies easy to introduce.
- Adding compensations means changes scatter across services.

### Orchestration (Central Coordinator)

A **saga orchestrator** explicitly calls each step and decides what to do on failure.

```
Orchestrator
   │ call inventory.reserve()
   │ ◄── reserved
   │ call payment.charge()
   │ ◄── failure
   │ call inventory.release()       ← compensation
   │ ◄── released
   │ publish OrderFailed
```

Implemented with: **Temporal** (or Cadence), **AWS Step Functions**, **Camunda**, **Netflix Conductor**, or hand-rolled state machines.

Pros:
- Workflow is explicit, version-able, debuggable.
- Centralized error handling.
- Easier to test end-to-end.
- Built-in retries, timeouts, signal handling.

Cons:
- Orchestrator is itself a service to operate.
- Couples services to the orchestrator's view of the world.
- Can become a "god class" if not carefully scoped.

### When to Pick Which

| Situation | Pick |
|-----------|------|
| 2–3 steps, simple linear flow | Choreography |
| > 4 steps, branches, retries | Orchestration |
| Need to visualize / audit the workflow | Orchestration |
| Each service team owns its own piece, low coordination | Choreography |
| Workflows changing weekly | Orchestration (one place to change) |

Many production systems mix: orchestrator for primary flow, events for side effects.

### Compensation Semantics

A compensation is **not a rollback** — it's a business-level undo.

```
Step                        Compensation
charge card                 refund (with original tx id)
reserve inventory           release reservation
send email                  send "actually, cancel that" email
```

The compensating action must be **idempotent** — it might run more than once (retries, replay). Track state explicitly (`reservation_released = true`) and no-op on repeat.

Some actions are not compensable: "send SMS to user" can't be unsent. Design for forward-only flow at those points, or move them after the point of no return.

### Idempotency Is Mandatory

Saga steps are at-least-once. Every step must dedup by saga ID + step ID:

```
service.reserve(orderId="o123", step="reserve_v1")
   if step already processed → return cached result
   else                       → do work, record step
```

Stripe, Temporal, Step Functions: all enforce this with deterministic step IDs.

### Saga State

Where the orchestrator (or aggregate of services in choreography) tracks progress.

```
saga_id        VARCHAR PK
current_step   ENUM
status         ENUM (in_progress, completed, failed, compensating)
context        JSONB        -- inputs, intermediate results
created_at     TIMESTAMPTZ
updated_at     TIMESTAMPTZ
```

Recovery: on orchestrator restart, load all `in_progress` sagas and resume.

### Saga + Outbox

To publish events atomically with a DB write, use the **Transactional Outbox pattern**:

```sql
BEGIN;
INSERT INTO orders (...);
INSERT INTO outbox (event_type, payload) VALUES ('OrderPlaced', '{...}');
COMMIT;
```

Separate process polls `outbox` (or CDC via Debezium) → publishes to Kafka. Atomic with the order write. No "we committed but the event didn't fire" bug.

See [Transactional Outbox](./outbox.md).

### Failure Modes

| Failure | Handling |
|---------|---------|
| Network blip during step | Retry with backoff; idempotency dedupes |
| Step permanently fails | Compensate prior steps; mark saga failed |
| Orchestrator crashes mid-flow | On restart, replay from last persisted step |
| Compensation itself fails | Retry; alert; possibly require human intervention |
| Saga stuck in `in_progress` | Timeout per step; force-fail and compensate |

### Visualizing

```
ORDER PLACED
   │
   ▼
[Reserve Inventory]──fail──►[Release Inventory]
   │ ok                       │
   ▼                          ▼
[Charge Card]──fail──►[Refund + Release Inventory]
   │ ok                       │
   ▼                          ▼
[Schedule Shipment]──fail──►[Refund + Release + cancel?]
   │ ok                       │
   ▼                          ▼
COMPLETED                  FAILED
```

Hand-drawing this diagram in an interview is half the answer.

### Common Mistakes

| Mistake | Fix |
|---------|-----|
| Treating saga as a transaction | It's eventually consistent — design UX around it |
| Steps not idempotent | Use deterministic step IDs + dedup table |
| Long-held locks across steps | Don't hold DB locks between saga steps — use optimistic versioning |
| One huge orchestrator owning many domains | Split per business workflow |
| No timeouts on steps | Stuck saga; always set a per-step deadline |
| Compensations not tested | Run failure-injection drills |

> [!NOTE]
> Sagas trade ACID for availability + service autonomy. The price: every business invariant becomes an explicit compensation. Design the compensations *first*; the happy path is the easy part.

### Interview Follow-ups

- *"Why not 2PC?"* — Blocks resources on participants; coordinator failure leaves locked rows; doesn't work over WAN.
- *"How do you implement Temporal-style workflows without Temporal?"* — Persistent state machine in DB + idempotent step handlers + scheduler. Re-inventing Temporal, badly.
- *"What about sagas with parallel branches?"* — Both choreography and orchestration support fan-out; orchestrator must wait for all branches before next step (`Promise.all` semantics).
