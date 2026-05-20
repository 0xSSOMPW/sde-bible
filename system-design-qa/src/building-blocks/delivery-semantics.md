# Q: At-most-once, at-least-once, exactly-once — what they mean and how to achieve them.

**Answer:**

Delivery semantics describe what happens when messages are sent across a network that may fail. They're not properties of brokers; they're properties of **producer + broker + consumer + dedup** working together.

### The Three Semantics

**At-most-once**
- Send and forget.
- Message may be lost.
- Never duplicated.
- Fastest.

**At-least-once**
- Retry until acknowledged.
- May duplicate.
- Never lost.
- Most common default.

**Exactly-once**
- Single observable effect.
- Hardest; requires producer + consumer + storage coordination.

### Achieving Each

**At-most-once**:

```
producer.send(msg)
# no retry on failure
# no ack required
```

OK for: telemetry where freshness > completeness; UDP-like protocols.

**At-least-once**:

```
producer.send(msg)
on ack timeout: retry with backoff
```

Plus:
- Broker durability (replication, fsync).
- Consumer commits offset AFTER processing.
- Idempotent processing.

This is what 99% of production systems use.

**Exactly-once**:

Two flavors:
- **Exactly-once delivery**: impossible over an unreliable network (Two Generals Problem).
- **Exactly-once effect** (idempotent): achievable; producer can duplicate, but consumer ensures the work happens once.

### How to Make Effects Exactly-Once

**1. Idempotent consumer.**

Consumer dedups by event ID before applying:

```sql
INSERT INTO processed_events (event_id) VALUES (?) ON CONFLICT DO NOTHING;
-- if conflict: skip
-- else: do the work
```

`event_id` from producer, immutable.

**2. Transactional consumer.**

Wrap "do work + commit offset" in one transaction:

```
begin
  apply event (idempotent OR uses event ID)
  commit consumer offset to same store
commit
```

Kafka EOS_V2: producer transactions + consumer reads `read_committed` to skip aborted records.

**3. Outbox + inbox pattern.**

Producer atomically writes outbox + business data. Consumer atomically writes inbox + applies effect.

See [Transactional Outbox](../patterns/outbox.md).

### Kafka's Exactly-Once Story

- **Idempotent producer** (`enable.idempotence=true`): broker dedups by `(producer_id, seq)`. Eliminates dupes within one producer session.
- **Transactions**: producer writes atomically to multiple topics + commits consumer offsets in one tx. Consumer with `isolation.level=read_committed` skips aborted records.
- **End-to-end**: app must use Kafka transactions correctly and read with `read_committed`.

Cost: ~20–40% throughput hit; small latency increase. Worth it for billing pipelines and similar.

### Common Cases

**Order placement webhook**:
- At-least-once + idempotency key.
- Processor dedups by key.
- "Exactly-once effect."

**Email sending**:
- At-least-once + dedup by `dedup_key`.
- Bouncing duplicate sends → just don't send the email again.

**Counters / analytics**:
- At-least-once + dedup by event_id at the aggregator.
- Or accept small overcount for higher throughput.

**Financial ledger**:
- At-least-once + idempotency key.
- Append-only ledger; duplicate inserts rejected by unique constraint.
- Reconciliation pass catches anything that slipped through.

### Where Each Layer Can Fail

```
producer ─send→ broker ─store→ consumer ─process→ effect ─commit→
```

A failure between any two stages requires the *next attempt* to be idempotent.

### Common Mistakes

| Mistake | Reality |
|---------|---------|
| "Exactly-once delivery" guaranteed by broker | Marketing; in practice = at-least-once + dedup |
| Trusting client clocks for dedup IDs | Generate IDs at producer service-side |
| Dedup table without retention | Grows forever; TTL old entries |
| Commit offset before processing finishes | Loss on crash |
| Process then commit, but processing isn't idempotent | Duplicates on crash → wrong state |

> [!NOTE]
> Stop asking for exactly-once delivery. Ask for exactly-once *effects* — that's achievable with idempotent operations and a dedup table. The semantics of the wire are at-least-once; the semantics of your business logic must be at-most-once on top.

### Interview Follow-ups

- *"How does Kafka achieve exactly-once across two topics?"* — Transactional producer wraps writes + offset commit; consumer reads `read_committed`. Atomic from the consumer's POV.
- *"What if the dedup store is down?"* — Fail-closed (reject the message) or fail-open (allow duplicate). Pick per business cost.
- *"Why is exactly-once delivery impossible?"* — Two Generals Problem: in an unreliable network, you can never know with certainty that your peer received the message. Retries are necessary; therefore duplicates are unavoidable; therefore consumer dedup is mandatory.
