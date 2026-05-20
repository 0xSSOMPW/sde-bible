# Q: Event Sourcing — when is "store every event" worth it?

**Answer:**

Event Sourcing stores **the sequence of events that produced state**, not the state itself. State is derived by replaying events. Powerful pattern with high ceremony — right for some domains (banking, audit), overkill for most.

### The Core Idea

Traditional model:

```
accounts table:
   id | balance
   42 | 1000
```

Event-sourced model:

```
events log:
   id=1  type=AccountOpened    account=42  balance=0
   id=2  type=Deposited        account=42  amount=500
   id=3  type=Deposited        account=42  amount=600
   id=4  type=Withdrawn        account=42  amount=100

derived: balance = 0 + 500 + 600 - 100 = 1000
```

The events are immutable. The balance is a **projection** derived by replaying them.

### Properties

- **Full audit trail.** Every change is a permanent record with who/when/why.
- **Time travel.** Replay to any point: "what was the balance on July 4?"
- **Multiple read models.** Same event log → many projections.
- **Reproducible bugs.** Bug in derivation? Replay from log with fixed code.
- **Natural CDC source.** Other systems can subscribe to the event stream.

### The Pieces

```
Commands ──► Aggregate ──► Events ──► Event Store ──► Projections / Read Models
                              │                              │
                              ▼                              ▼
                       publish to bus              query views (CQRS)
```

- **Command**: intent ("transfer $100").
- **Aggregate**: domain object validating the command + emitting events.
- **Event**: fact ("transferred $100"). Past tense, immutable.
- **Event Store**: append-only log keyed by aggregate ID.
- **Projection**: process consuming events to build read views.

### Aggregate Pattern

```python
class Account:
    def __init__(self, events):
        self.balance = 0
        self.id = None
        for e in events:
            self.apply(e)

    def apply(self, event):
        if event.type == "AccountOpened":
            self.id = event.account
        elif event.type == "Deposited":
            self.balance += event.amount
        elif event.type == "Withdrawn":
            self.balance -= event.amount

    def withdraw(self, amount):
        if amount > self.balance:
            raise InsufficientFunds()
        return Event("Withdrawn", account=self.id, amount=amount)
```

Load: read events, apply. Command: emit event. Save: append event.

### Snapshots

Replaying every event from year zero gets slow at high counts. Snapshot the aggregate state every N events:

```
load(account_id):
    snapshot = load_latest_snapshot(account_id)
    events = load_events_after(snapshot.version)
    return apply_all(snapshot, events)
```

Snapshots are an optimization, not the source of truth. Throw them away anytime; they regenerate.

### Schema Evolution

Events are forever. You can't change one — only add new ones. Strategies:

- **Add new field with default** (Avro/Proto backward-compatible).
- **Upcasters**: code that rewrites old event versions to new shapes at load time.
- **Never delete**. Mark "deprecated" instead.

A subtle bug: a 2017 event of type `AddressUpdated` won't have a `country` field you added in 2023. Your replay code must handle that.

### When Event Sourcing Is Right

- Compliance / audit-heavy domains (banking, healthcare, legal).
- "What changed and when" is a primary query.
- Domain has clear, stable business events (already DDD-modeled).
- You want eventual derivation of multiple read models.
- Replay-based recovery is valuable.

### When It's Wrong

- Pure CRUD ("update users.email"); events would be `EmailChanged` and nothing else interesting.
- Team unfamiliar with the pattern.
- High write throughput on a single aggregate (every event is a serialization point).
- Reporting needs ad-hoc SQL across all entities at once (projections rescue this, but with effort).

### Implementation Options

| Stack | Notes |
|-------|-------|
| Postgres + outbox + Kafka | Simplest; events in a table, indexed by aggregate_id |
| Kafka + KTable / state stores | Logs are first-class; replay is natural |
| EventStoreDB | Purpose-built; tooling for projections, snapshots, replay |
| Axon Framework | JVM-first event sourcing framework |
| Marten (Postgres) | Postgres-as-event-store; .NET-focused |

### Eventual Consistency

Same gotcha as CQRS. Projections lag the event store. Read-your-writes patterns:

- Optimistic UI.
- Query the latest snapshot + recent events.
- Pinned read after write for short window.

### Concurrency Control

Two commands on the same aggregate race. Use **optimistic concurrency**:

```
load events at version 42
build aggregate state
emit new event(s)
append IF current version == 42
   on mismatch: reload + retry
```

Optimistic version checks at append; conflicts trigger retry. No locks held.

### Sagas Often Live Above Event Sourcing

Inter-aggregate workflows: events from one aggregate trigger commands on another. The orchestrator (or choreography) lives outside the aggregate boundary. See [Saga](./saga.md).

### Cost of Event Sourcing

- **Storage**: O(events) forever. 10× more than current-state storage; with compression, still ~3×.
- **Cognitive load**: every change starts with "what event is this?"
- **Migration**: schema versioning is non-trivial.
- **Tooling**: rebuild from offset 0 is your safety net; needs to actually work.

### Patterns It Enables

- **Time-travel debugging**: replay to the exact moment a bug appeared.
- **A/B projection**: build two read models from same events for migration.
- **Audit trail**: free, complete, queryable.
- **Temporal queries**: "what did this look like at midnight last Friday?"
- **External integration**: external system gets a clean event stream, no DB CDC kludges.

### Common Mistakes

| Mistake | Reality |
|---------|---------|
| Treating events as "what UI needs" | Events are *business facts*; UI consumes projections |
| Mutable events | Defeats the entire pattern; events are immutable |
| Skipping snapshots | Replay grows linearly; eventually unusable |
| Single global event stream | Hot partition; partition by aggregate type or ID |
| Forgetting upcasters | Old events fail to deserialize when schema evolves |

> [!NOTE]
> Event Sourcing is the strongest, slowest version of "audit-ability." If the domain doesn't need it strongly, a normal DB + outbox is usually a better trade.

### Interview Follow-ups

- *"Difference between Event Sourcing and CDC?"* — ES emits *business events* from the domain layer (intent + meaning). CDC emits *database-row changes* (after the fact). ES has semantics; CDC has bytes.
- *"How do you delete data (GDPR)?"* — Crypto-shredding: encrypt PII with a per-user key, delete the key on request → data unrecoverable. Or null out specific fields in old events ("tombstone").
- *"How do you replay 100M events?"* — Snapshot + replay only after; parallelize per aggregate; back-pressure the projection rebuilder.
