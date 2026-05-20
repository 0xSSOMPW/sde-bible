# Q: Design a payment system (Stripe-like).

**Answer:**

Money-moving system. ACID and idempotency are non-negotiable. The interview tests **double-entry bookkeeping**, **idempotency**, **external integrations** (card networks, banks), and **reconciliation**.

### Requirements

**Functional**:
- Authorize a card / payment method.
- Capture / refund.
- Multi-currency.
- Subscriptions / recurring.
- Disputes & chargebacks.
- Payouts to merchants.

**Non-functional**:
- Strict consistency: no double-charge, no lost charge.
- Audit trail mandatory (regulatory).
- PCI compliance (handle card data with care).
- Provider outages (Visa/Mastercard, banks) tolerated.

### High-Level Architecture

```
Merchant API ────► Payment Gateway ────► Card Network / Bank / Wallet
                       │
                       ▼
                  Ledger Service (double-entry)
                       │
                       ▼
                  Postgres (immutable ledger)
                       │
                       ▼
                  Reconciliation jobs
                       │
                       ▼
                  Payout Service → Bank ACH/SEPA
```

### Ledger — The Core

Double-entry bookkeeping: every transaction has matching debit + credit entries.

```sql
CREATE TABLE ledger_entries (
    id              BIGSERIAL PRIMARY KEY,
    txn_id          UUID NOT NULL,           -- groups paired entries
    account_id      BIGINT NOT NULL,
    amount          NUMERIC(18, 4) NOT NULL, -- positive or negative
    currency        CHAR(3) NOT NULL,
    type            TEXT NOT NULL,           -- 'authorization', 'capture', 'refund', ...
    posted_at       TIMESTAMPTZ DEFAULT now(),
    metadata        JSONB
);

CREATE INDEX ON ledger_entries (txn_id);
CREATE INDEX ON ledger_entries (account_id, posted_at);
```

**Immutable**: never UPDATE or DELETE. Corrections = compensating entries.

For each transaction, total = 0 (debits + credits balance). Enforced by check:

```
SELECT txn_id, sum(amount) FROM ledger_entries GROUP BY txn_id HAVING sum(amount) != 0;
-- must return zero rows; otherwise inconsistency
```

### Charge Lifecycle

```
1. Authorize:    hold $100 on card
                 ledger: customer_holds += 100, customer_funds -= 100
2. Capture:      actually take the money
                 ledger: customer_funds_held -= 100, merchant_pending += 100
3. Payout:       transfer to merchant bank
                 ledger: merchant_pending -= 100, bank_outflow += 100
4. (or) Refund:  ledger: undo earlier entries
```

State machine on `charges` table tracks current lifecycle stage.

### Idempotency

Critical. Every external API call uses `Idempotency-Key`:

```
POST /v1/charges
Idempotency-Key: abc123-...
{
  "amount": 100, "currency": "USD",
  "source": "tok_xyz", "description": "..."
}
```

Server stores the result keyed by `(merchant_id, idempotency_key)`. Replay returns cached response — no double-charge.

See [Idempotent REST APIs](../../java-qa/src/spring/idempotency.md).

### Concurrency on Hot Accounts

A merchant account might receive 1000 charges/sec. Naive row-level lock would serialize.

Approaches:
1. **Optimistic** (`@Version` / version column) — retry on conflict; fine for moderate contention.
2. **Sharded balance** — N sub-accounts; pick one randomly; aggregate periodically.
3. **Append-only ledger + materialized balance** — never lock; recompute balance lazily or via a stream.

The last is what most modern ledger systems do (Square, Modern Treasury). No row-locks on the hot path.

### External Provider Integration

Card networks, ACH, bank APIs. Properties:
- Slow (100s of ms).
- Eventually consistent (webhooks notify final state).
- Can fail in many ways (decline, network, fraud).

Pattern:
```
POST /charges
   ├── INSERT pending charge (DB, atomic with idempotency record)
   ├── async call provider
   ├── return 200 with pending state
   └── webhook from provider → update state machine
```

Never call provider inside a DB transaction.

### Reconciliation

Daily job compares your ledger to provider's statements:

```
For each charge:
  Compare: our_record.captured_amount vs provider_settlement.amount
  Flag mismatches for investigation.
```

Discrepancies happen — provider downtime, network drops, edge-case fees. Reconciliation catches them.

### Disputes & Chargebacks

Bank initiates a chargeback months after the original charge. Steps:
- Webhook from provider → create `dispute` record.
- Merchant has X days to submit evidence.
- Final ruling: funds returned to customer (`refund` ledger entries) or upheld.

Money is reversible until ~120 days post-charge. Plan reserve accounts accordingly.

### Multi-Currency

Store amount in **smallest currency unit** (`amount INT in cents`):
```
$1.00 → 100
¥10 → 10 (yen has no decimal)
```

Never store as floating-point. `NUMERIC` in Postgres or scaled int.

Conversion: lock the FX rate at the time of charge; store `(amount, currency, rate_used)`.

### Subscriptions / Recurring

```
subscriptions (id, customer_id, plan_id, status, next_billing_at)
```

Cron job picks subscriptions whose `next_billing_at < now()`. Charge them. Update next.

Failure handling: retry with backoff, eventually mark `past_due`, then cancel. Email customer at each step.

### Payouts to Merchants

Periodic (daily/weekly) sweep:
- Sum merchant's pending balance.
- ACH/SEPA transfer.
- Decrement pending, increment paid_out in ledger.

Hold period (3–7 days) before payout to cover disputes.

### Storage Picks

| Data | Store |
|------|-------|
| Ledger | Postgres (single primary, replicated; ACID) |
| Charges, customers, subscriptions | Postgres |
| Idempotency cache | Postgres or Redis with backup |
| Audit log of API calls | Append-only Postgres or Kafka → S3 |
| Reporting / dashboards | ClickHouse, materialized from ledger |

Postgres at the center because ACID. Distributed SQL (Spanner, CockroachDB) for global single-source-of-truth ledger.

### PCI Compliance

- **Never store raw card numbers** unless PCI Level 1 certified.
- Tokenize via provider: card → opaque `tok_xyz`; only the provider sees PAN.
- Limit access logs to who handled tokens.

### Fraud / Risk

- Real-time scoring: device fingerprint, IP, velocity, history.
- ML model output: allow / review / reject.
- Off-the-shelf: Stripe Radar, Sift, Riskified.

### Failure Modes

| Failure | Handling |
|---------|---------|
| Provider 5xx | Retry with backoff; idempotency dedups |
| DB primary down | Block writes; promote replica |
| Ledger inconsistency detected | Halt new charges; investigate; correcting entries |
| Wrong amount captured | Issue refund + new capture; never edit historical entries |

### Multi-Region

Money systems are typically single-primary even when global. Latency added for non-primary regions, but the trade is **correctness** > latency for money.

Distributed SQL (Spanner / Cockroach) lets you have linearizable global writes for a latency cost.

### Common Mistakes

| Mistake | Reality |
|---------|---------|
| Float for money | Rounding loss; use NUMERIC / int cents |
| No idempotency key | Duplicate charges on retry |
| Updates to ledger rows | Lose audit trail; use append-only |
| Single API for charge + capture + payout | Different lifecycles; split services |
| Trust your DB lag for reconciliation | Periodic exact reconciliation against provider |

> [!NOTE]
> Two non-negotiables in payments: **idempotency on every external mutation** and **immutable ledger as source of truth**. Get those right; everything else is plumbing.

### Interview Follow-ups

- *"How do you handle 'we charged but customer says they didn't get the service'?"* — Dispute workflow → temporary hold of funds → evidence → resolution. Recorded in ledger throughout.
- *"How do you scale across regions while keeping correctness?"* — Single primary DB or distributed SQL with cross-region linearizability. Accept latency; never accept eventual consistency for money.
- *"How do you test payment systems safely?"* — Provider sandbox environments; test mode credentials; shadow ledger that doesn't move real money.
