# Q: Design a Reward / Loyalty Points & Cashback Service (Paytm / PhonePe / Amazon Pay rewards, airline miles, Starbucks stars, Dream11 coins).

**Answer:**

A rewards service looks like "add N to a counter" but is really **a money-adjacent ledger plus a rules engine plus a budget controller**. Points are a **liability on the company balance sheet** (IFRS 15 / Ind AS 115 deferred revenue), users treat them as cash, and fraudsters mine them relentlessly. Every design decision should be justified by: *can we prove why this user has this balance, and can we cap what we owe?*

The interview tests: **event-driven earn with an idempotency key per source event**, **immutable double-entry points ledger (balance is a projection)**, **hold / commit / release redemption**, **FIFO expiry buckets**, **reversal and clawback on refund**, **probabilistic rewards (scratch cards) under a hard budget**, **fraud and referral abuse**, **liability accounting and reconciliation**, and **spike handling (IPL final, Diwali, Big Billion Days)**.

### Requirements

**Functional**:
- Earn points / cashback from events: order placed, payment succeeded, bill paid, referral completed, KYC done, daily check-in, streak milestone.
- Configurable **campaigns and rules** (no deploy): "2x points on groceries this weekend, cap 500/day, Gold tier gets 1.5x on top".
- Points **balance, history, and expiry** views (user and support console).
- **Redeem** points against orders / bills / vouchers / bank transfer (hold, commit, release).
- **Reverse** on refund, cancellation, chargeback, fraud (clawback; balance may go negative).
- **Scratch cards / spin wheels / lucky draws** with probabilistic payouts.
- **Tiers and streaks** (Silver / Gold / Platinum; 7-day streak bonus).
- Referral rewards (referrer and referee) with anti-abuse.
- Notifications: "you earned", "expiring in 7 days", "tier upgraded".
- Finance: per-campaign budget, liability reports, GL export, reconciliation.

**Non-functional**:
- **Never double-credit or double-debit**: exactly-once *effect* on balance.
- Earn is async and may lag by seconds; **redemption is synchronous and strongly consistent**.
- Redeem p99 < 100 ms; balance read p99 < 30 ms; earn event-to-credit p95 < 5 s.
- Handle 10x spikes (IPL final, flash sale) without losing or duplicating events.
- Immutable audit trail retained 7-10 years; explain-any-balance capability.
- Budget never exceeded by more than a small bounded overshoot.
- GDPR / DPDP: export and erase personal data without breaking the ledger.

**Out of scope**: card-issuer co-brand loyalty, coalition partner settlement contracts (mention as extension), voucher inventory procurement.

### Capacity Estimation

- 100M registered users, 30M DAU; 20M orders/day + 40M payment events/day.
- Each business event matches ~1.5 rules on average, so about **90M earn decisions/day** = ~1k/sec avg, **~15-20k/sec peak** (IPL final: 10x for 30 minutes).
- Ledger entries: each earn = 2 entries (user liability credit, campaign expense debit); redeem = 2-4; expiry = 2. About **250M entries/day** = ~3k/sec avg, ~40k/sec peak. At ~150 B per row = ~40 GB/day; partition by month, archive to S3 Parquet after 13 months.
- Balance reads: 30M DAU x 5 views = 150M/day = ~1.7k/sec avg, 10k/sec peak. Served from Redis projection.
- Scratch cards: 20M issued/day, ~12M scratched; each scratch needs one budget decision = ~150/sec avg, ~3k/sec peak.
- Active buckets (points lots): 100M users x ~6 live lots = 600M rows x 100 B = **60 GB**: shard by `user_id`.
- Kafka: ~100M events/day x 1 KB = 100 GB/day, 7-day retention = 700 GB; 64 partitions per topic keyed by `user_id`.

Writes are naturally partitioned by `user_id` (no global hot row) **except** campaign budget counters, which are the real hot spot.

### API Design

```
POST /v1/rewards/events                 (internal; or consume from Kafka instead)
{ "event_id": "ord_9981:payment_success", "user_id": "u_42", "type": "ORDER_PAID",
  "amount": 1499.00, "currency": "INR", "attrs": {"category":"GROCERY","merchant":"m_7","city":"BLR"},
  "occurred_at": "2026-10-26T08:00:00Z" }

GET  /v1/users/{id}/balance             -> { "points": 4120, "pending": 200, "expiring_30d": 350, "tier": "GOLD" }
GET  /v1/users/{id}/ledger?cursor=...   -> paginated entries with reason codes

POST /v1/redemptions/hold               Idempotency-Key: <order_id>:redeem
{ "user_id": "u_42", "points": 500, "order_id": "ord_9982", "ttl_s": 900 }
   -> 201 { "hold_id": "h_1", "expires_at": "..." }   | 409 INSUFFICIENT_POINTS
POST /v1/redemptions/{hold_id}/commit   -> 200 { "txn_id": "t_77" }
POST /v1/redemptions/{hold_id}/release  -> 200

POST /v1/rewards/reversals              { "source_event_id": "ord_9981:payment_success", "reason": "REFUND", "ratio": 1.0 }

GET  /v1/scratch-cards?user_id=...      -> list of unscratched cards
POST /v1/scratch-cards/{id}/scratch     -> { "reward": {"type":"CASHBACK","value":25} }

Admin:
POST /v1/campaigns                      { rule DSL JSON, budget, window, status: DRAFT }
POST /v1/campaigns/{id}/simulate        -> dry-run on replayed last-7-day events: cost, hit rate
POST /v1/campaigns/{id}/activate
```

Key design points: events carry a **globally unique `event_id`** supplied by the upstream (never generated by us), the redeem endpoints take an **Idempotency-Key**, and campaign activation requires **simulate first**.

### High-Level Architecture

```
 Order Svc ─┐                                          ┌─► Notification Svc ─► push/SMS/email
 Payment Svc ├─► outbox ─► Kafka (events.*) ─► Earn Consumers ─► Rules Engine
 Referral ──┘                                    │            (campaign cache, tier, caps)
                                                 │                    │
                                                 │                    ▼
                                                 │            Budget Controller ◄─ Redis (token buckets)
                                                 │                    │
                                                 ▼                    ▼
                                          Fraud / Risk ───► Points Ledger Service ──► Postgres shards
                                                                      │                (entries, buckets,
 Checkout ─► Redemption API (hold/commit/release) ───────────────────┤                 holds, idem keys)
                                                                      │
                                                          outbox ─► Kafka (ledger.*)
                                                                      │
                  ┌───────────────────┬──────────────────┬───────────┴──────────┐
                  ▼                   ▼                  ▼                      ▼
          Balance Projection   Analytics / ClickHouse   Finance GL Export   Notification triggers
          (Redis + PG)         (campaign ROI)           + Reconciliation    (expiry, tier)
```

Two paths with different guarantees:
- **Earn path**: asynchronous, at-least-once from Kafka, made **effectively exactly-once** by a unique constraint on `(source_event_id, rule_id)`.
- **Redemption path**: synchronous, strongly consistent on the user's shard, using a conditional balance check inside one DB transaction.

### Data Model

```sql
-- Immutable double-entry ledger. One business operation = one txn_id, entries sum to zero.
CREATE TABLE points_entries (
    entry_id      BIGSERIAL,
    txn_id        UUID        NOT NULL,
    account_id    TEXT        NOT NULL,  -- 'user:u_42:points' | 'campaign:c_9:expense' | 'liability:points' | 'expired:pool'
    delta         BIGINT      NOT NULL,  -- signed points (cashback stored in minor units, separate unit column)
    unit          TEXT        NOT NULL,  -- 'POINTS' | 'INR_PAISE'
    kind          TEXT        NOT NULL,  -- EARN | HOLD | COMMIT | RELEASE | EXPIRE | REVERSE | ADJUST | CONVERT
    source_event  TEXT        NOT NULL,  -- upstream event id: idempotency anchor
    rule_id       TEXT,
    campaign_id   TEXT,
    bucket_id     BIGINT,                -- which lot was touched
    created_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    meta          JSONB,
    PRIMARY KEY (entry_id, created_at)
) PARTITION BY RANGE (created_at);

-- Idempotency: the same source event + rule + kind can never post twice
CREATE UNIQUE INDEX uq_entry_idem ON points_entries (source_event, COALESCE(rule_id,''), kind, account_id, created_at);
-- (production variant: a separate small idem table keyed (source_event, rule_id, kind) avoids including the partition key)

-- Lots of points with their own expiry: needed for FIFO burn and expiry.
CREATE TABLE points_buckets (
    bucket_id     BIGSERIAL PRIMARY KEY,
    user_id       TEXT   NOT NULL,
    earned        BIGINT NOT NULL,
    remaining     BIGINT NOT NULL CHECK (remaining >= 0),
    held          BIGINT NOT NULL DEFAULT 0 CHECK (held >= 0 AND held <= remaining),
    expires_at    TIMESTAMPTZ NOT NULL,
    source_event  TEXT NOT NULL,
    campaign_id   TEXT,
    version       INT NOT NULL DEFAULT 0
);
CREATE INDEX ON points_buckets (user_id, expires_at) WHERE remaining > 0;
CREATE INDEX ON points_buckets (expires_at) WHERE remaining > 0;     -- expiry sweeper

-- Balance projection (cache of SUM(entries)); can always be rebuilt.
CREATE TABLE user_balance (
    user_id TEXT PRIMARY KEY, available BIGINT NOT NULL, held BIGINT NOT NULL,
    lifetime_earned BIGINT NOT NULL, tier TEXT, last_entry_id BIGINT, updated_at TIMESTAMPTZ
);

CREATE TABLE redemption_holds (
    hold_id UUID PRIMARY KEY, user_id TEXT, order_id TEXT UNIQUE, points BIGINT,
    state TEXT CHECK (state IN ('HELD','COMMITTED','RELEASED','EXPIRED')),
    allocation JSONB,              -- [{bucket_id, points}] so release is exact
    expires_at TIMESTAMPTZ, created_at TIMESTAMPTZ
);

CREATE TABLE campaigns (
    campaign_id TEXT PRIMARY KEY, rule JSONB NOT NULL, version INT, status TEXT,
    starts_at TIMESTAMPTZ, ends_at TIMESTAMPTZ,
    budget_total BIGINT NOT NULL, budget_spent BIGINT NOT NULL DEFAULT 0, owner TEXT, cost_center TEXT
);

CREATE TABLE outbox (id BIGSERIAL PRIMARY KEY, topic TEXT, key TEXT, payload JSONB, created_at TIMESTAMPTZ, published BOOLEAN DEFAULT false);
```

**Immutable**: no UPDATE/DELETE on `points_entries`. Buckets and `user_balance` are *mutable projections* that must always reconcile to the ledger.

### Earn Rules Engine

Rules are **data, not code**, so marketing can ship campaigns without a release. A compact JSON DSL evaluated against the event:

```json
{
  "rule_id": "grocery_weekend_2x",
  "campaign_id": "c_diwali_26",
  "when": { "all": [
      { "event": "ORDER_PAID" },
      { "attr": "category", "in": ["GROCERY", "FRESH"] },
      { "amount": { "gte": 299 } },
      { "time": { "dow": ["SAT","SUN"], "tz": "Asia/Kolkata" } },
      { "user": { "segment": "NOT_FRAUD_FLAGGED" } }
  ]},
  "reward": { "type": "POINTS", "formula": "floor(amount * 0.01) * multiplier", "multiplier": 2.0,
              "expiry_days": 90 },
  "caps": { "per_txn": 200, "per_user_day": 500, "per_user_campaign": 2000, "per_campaign_budget": 50000000 },
  "stacking": { "group": "base_earn", "mode": "MAX" },
  "priority": 10,
  "window": { "start": "2026-10-24T00:00:00+05:30", "end": "2026-11-02T23:59:59+05:30" }
}
```

Evaluation pipeline per event:
1. **Candidate lookup**: index active campaigns by `event_type` (and merchant/category) in an in-memory cache refreshed by a campaign-changed Kafka topic; evaluating 5k rules per event is a non-starter.
2. **Predicate evaluation**: compile JSON to an AST once (e.g. CEL, JsonLogic, or Drools for heavy shops); evaluation is pure and side-effect free.
3. **Stacking resolution**: rules in the same `stacking.group` resolve by `MAX` / `SUM` / `FIRST_BY_PRIORITY`. Base earn + tier multiplier + campaign bonus are different groups so they stack; two overlapping campaigns on the same base usually take MAX.
4. **Caps**: per-user counters in Redis/DB keyed by `(user, campaign, window)`; the cap check and the ledger write must agree (see below).
5. **Budget reserve**: call the Budget Controller (below).
6. **Emit grant** to the Ledger Service with `(source_event, rule_id)` idempotency key.

Properties of a good engine:
- **Deterministic and replayable**: same event + same rule version = same reward. Store `rule_version` in entry meta; this lets support answer "why did I get 40 points?" and lets finance replay.
- **Versioned and immutable rules**: editing creates version N+1; in-flight events use the version active at `occurred_at`, not processing time.
- **Simulate before activate**: replay last 7 days of events through the draft rule and show projected cost, top-10 earners, and cap hit rate. This catches the "accidentally 100x" campaign before it bankrupts the budget.
- **Kill switch**: per-campaign `status = PAUSED` propagates via cache invalidation in < 5 s.

> [!NOTE]
> **Event time, not processing time.** A consumer lag of 20 minutes during IPL must not push an order placed at 23:58 on the last day of a campaign into "campaign expired". Evaluate windows against `occurred_at`, and accept events up to a grace period late.

### Idempotent Earn (Exactly-Once Effect)

Kafka gives at-least-once. We get exactly-once **effect** by making the write idempotent at the database:

```
consume event E (offset o)
BEGIN
  for each matched grant g:
     INSERT INTO idem(source_event, rule_id, kind) VALUES (E.id, g.rule, 'EARN')
         ON CONFLICT DO NOTHING          -- 0 rows => duplicate, skip
     if inserted:
        INSERT ledger entries (user +N, campaign expense -N)   -- same txn_id, sums to 0
        INSERT points_buckets (earned=N, remaining=N, expires_at)
        UPDATE user_balance SET available = available + N
        INSERT outbox (ledger.earned ...)
COMMIT
commit Kafka offset (after DB commit)
```

Crash before offset commit means redelivery, and the unique key turns it into a no-op. Crash after DB commit but before offset commit: same. The upstream `event_id` must be **stable across retries** (`order_id:payment_success`, not a random UUID per publish).

```java
public final class PointsLedger {
    private final DataSource ds;

    public record Grant(String sourceEvent, String ruleId, String userId,
                        long points, String campaignId, Instant expiresAt) {}

    /** Returns true if applied, false if it was a duplicate. */
    public boolean earn(Grant g) throws SQLException {
        try (Connection c = ds.getConnection()) {
            c.setAutoCommit(false);
            try {
                // 1. idempotency gate: PK on (source_event, rule_id, kind)
                try (PreparedStatement ps = c.prepareStatement(
                        "INSERT INTO idem(source_event, rule_id, kind) VALUES (?,?, 'EARN') " +
                        "ON CONFLICT DO NOTHING")) {
                    ps.setString(1, g.sourceEvent()); ps.setString(2, g.ruleId());
                    if (ps.executeUpdate() == 0) { c.rollback(); return false; }
                }
                UUID txn = UUID.randomUUID();
                // 2. double entry: user liability up, campaign expense down; sums to zero
                insertEntry(c, txn, "user:" + g.userId() + ":points", +g.points(), "EARN", g);
                insertEntry(c, txn, "campaign:" + g.campaignId() + ":expense", -g.points(), "EARN", g);
                // 3. FIFO lot for later burn / expiry
                try (PreparedStatement ps = c.prepareStatement(
                        "INSERT INTO points_buckets(user_id, earned, remaining, expires_at, source_event, campaign_id) " +
                        "VALUES (?,?,?,?,?,?)")) {
                    ps.setString(1, g.userId()); ps.setLong(2, g.points()); ps.setLong(3, g.points());
                    ps.setTimestamp(4, Timestamp.from(g.expiresAt()));
                    ps.setString(5, g.sourceEvent()); ps.setString(6, g.campaignId());
                    ps.executeUpdate();
                }
                // 4. projection + outbox in the SAME transaction
                try (PreparedStatement ps = c.prepareStatement(
                        "INSERT INTO user_balance(user_id, available, held, lifetime_earned) VALUES (?,?,0,?) " +
                        "ON CONFLICT (user_id) DO UPDATE SET available = user_balance.available + EXCLUDED.available, " +
                        "lifetime_earned = user_balance.lifetime_earned + EXCLUDED.lifetime_earned")) {
                    ps.setString(1, g.userId()); ps.setLong(2, g.points()); ps.setLong(3, g.points());
                    ps.executeUpdate();
                }
                insertOutbox(c, "ledger.earned", g.userId(), g);
                c.commit();
                return true;
            } catch (SQLException e) { c.rollback(); throw e; }
        }
    }

    // insertEntry(...) and insertOutbox(...) are plain parameterised INSERTs
}
```

### Double-Entry Ledger: Balance as Projection

Why double-entry for loyalty? Because points are a **liability**: every credit to a user must be matched by a debit to a funding account (campaign expense / marketing budget / referral pool). Finance can then sum `liability:points` and get outstanding obligation directly.

```
EARN 100 pts (campaign c_9):     user:u_42:points      +100
                                 campaign:c_9:expense  -100
REDEEM 100 pts (commit):         user:u_42:points      -100
                                 redeemed:pool         +100      (revenue-discount / settlement account)
EXPIRE 40 pts:                   user:u_42:points      -40
                                 expired:pool          +40       (breakage income)
REVERSE (refund) 100 pts:        user:u_42:points      -100
                                 campaign:c_9:expense  +100      (budget refunded)
```

Rules:
- Entries are **append-only**; corrections are compensating entries.
- `SUM(delta) GROUP BY txn_id = 0`, checked continuously by a verifier job.
- `user_balance.available == SUM(entries WHERE account = user:X:points) - held` is the **core invariant**; the projection is rebuilt from the ledger on drift.
- Cashback in rupees uses the same structure with `unit = INR_PAISE` and posts to the real cash ledger / wallet (see [Payment System](./payment-system.md)); never mix units in one account.
- Use integers only; fractional points are a rounding bug waiting to happen. Define rounding once (`floor` on earn, `ceil` on burn requirement) and document it.

### Redemption: Hold, Commit, Release

Redemption is a **two-phase** protocol because checkout can fail after the user chose to burn points.

```
Checkout                      Rewards Service                    DB (user shard)
   │ hold(500, order_9982) ───►  BEGIN
   │                              SELECT buckets FOR UPDATE ordered by expires_at   (FIFO, soonest first)
   │                              allocate 500 across buckets; held += x
   │                              user_balance: available -= 500, held += 500
   │                              INSERT hold(state=HELD, allocation=[...], expires=+15m)
   │ ◄── hold_id ──────────────  COMMIT
   │  ... payment succeeds ...
   │ commit(hold_id) ────────►   hold HELD->COMMITTED; buckets remaining -= x, held -= x;
   │                              ledger: user -500, redeemed:pool +500 ; outbox
   │  ... or payment fails / user abandons ...
   │ release(hold_id) ───────►   hold HELD->RELEASED; held -= x; available += 500 (exact buckets)
   (timer)  hold TTL elapses ─►   sweeper releases: state EXPIRED
```

Details that matter:
- **Idempotency**: `hold` keyed by `order_id` (UNIQUE); a retried hold returns the same `hold_id`. `commit` and `release` are state-machine transitions guarded by `WHERE state = 'HELD'`; repeat calls are no-ops.
- **Lock order**: lock `user_balance` row first, then buckets by `expires_at`; same order everywhere avoids deadlocks.
- **No lost points on timeout**: if `commit` times out at the caller, the caller retries (idempotent); if the caller never returns, the TTL sweeper releases. The commit-after-expiry race is resolved by the state check: expired hold means commit fails and checkout must re-hold.
- **Partial redemption**: points cover `min(balance, order_cap)`; a per-order cap (e.g. max 50% of cart) is a business rule enforced in the redemption service, not the client.
- **Conversion**: redeem rate (`1 point = ₹0.25`) is stored on the hold and ledger entry so a later rate change does not alter history.

> [!NOTE]
> Prefer **burning soonest-expiring points first (FIFO by expiry)**. It is better for the user (less waste) but *worse* for the company's breakage. This is a policy decision. Make it explicit and configurable per program; regulators in some regions mandate user-favourable ordering.

### Expiry: FIFO Buckets, Lazy vs Batch

Every earn creates a **bucket** (lot) with its own `expires_at`. A single running balance cannot expire correctly (what part of 1000 is old?); buckets can.

**Lazy expiry** (compute on read / on burn):
- Balance = `SUM(remaining) WHERE expires_at > now()`.
- Burn skips expired buckets.
- Pros: no sweeper load. Cons: liability reports overstate until swept; expiry ledger entries are never written, so finance cannot see breakage; "expiring soon" notifications need a separate query.

**Batch expiry** (sweeper):
- Nightly (or hourly, per timezone) job: `SELECT ... FROM points_buckets WHERE expires_at <= now() AND remaining > 0 LIMIT 5000`, for each: post `EXPIRE` ledger entries (user -N, expired:pool +N) and zero `remaining`. Idempotent via `(bucket_id, 'EXPIRE')`.
- Pros: ledger and liability are accurate; clean breakage numbers. Cons: write amplification at midnight.

**Production answer: both.** Reads and burns are lazy-correct (never use expired points, even if the sweeper is late), and the sweeper makes the ledger authoritative afterwards. Spread the load: expiry is evaluated at **end-of-day in user's timezone**, and buckets are rounded to `expires_at = end of month` so sweeps are predictable and sharded by `expires_at` partition rather than a thundering herd at 00:00:00 UTC.

Held points are **not** expired mid-hold: the sweeper skips `held > 0` buckets and re-evaluates after release (or extends by the hold TTL).

Extension policies (e.g. "any activity extends all expiry by 12 months", as with airline programs) are implemented as a *bucket update with a ledger note*, never by editing history silently.

### Reversal, Refund and Clawback

When an order is refunded, cancelled, or charged back, the points earned from it must be reversed.

```
refund event (order O, ratio 0.4)
  ├─ find original EARN entries by source_event = "O:payment_success" (all rules)
  ├─ reverse = round_down(earned * ratio)               -- partial refund = partial reversal
  ├─ idempotency key: (refund_id, rule_id, 'REVERSE')
  ├─ consume from the *same buckets* if still present (remaining -= x)
  ├─ if user already spent them  -> balance goes NEGATIVE (debt) instead of failing
  └─ ledger: user -x, campaign:c +x     (budget returned to campaign if still active)
```

Handling **negative balance**:
- Allow `available < 0` only through the reversal path (CHECK constraint exempts `kind = REVERSE`); redemption path still requires `available >= amount`.
- A user in debt cannot redeem until future earns net it out (earn first pays down debt, FIFO).
- Alternatively **block clawback and flag for manual recovery** when the amount is small; cheaper than angering users for 3 points.
- If the redeemed points were already converted into a voucher or bank payout, reversal creates a **receivable** on the finance books instead of a negative balance.

Reversal vs. un-earn ordering problem: refund event can arrive **before** the earn event (different services, different partitions). Handle it with a *tombstone*: record `REFUNDED(order O)`; when the earn arrives later, it is created already-reversed or skipped, and finance sees both entries.

Refund-after-redemption on the **spend** side (user paid with points, order cancelled) is a **release/refund of the commit**: create a new bucket restoring the points with the **original expiry** (or a minimum 30-day grace) so the user is not punished by cancellation.

### Cashback and Points Conversion; Multi-Currency

- **Points vs cashback**: points are a closed-loop currency (valued by program rules, devaluable); cashback is real money in a wallet (regulated, e-money rules, KYC limits). Keep them on **separate ledgers/units**; conversion is an explicit transaction (`CONVERT`: points -X, wallet +Y) at a stored rate.
- **Rate table**: `points_rate(program, currency, effective_from, rate)`; the rate used is stored on each redemption; devaluation applies to future conversions only (and may require notice by law).
- **Multi-currency**: earn computed on **minor units** in the transaction currency, converted to points via per-region rate. Store `(amount, currency, fx_rate, points)` in entry meta for audit; no floats.
- **Partner/coalition programs** (airline miles <-> bank points): treat as external accounts; transfers are two-leg with a settlement ledger and daily netting.

### Scratch Cards and Probabilistic Rewards

A scratch card is a **lottery with a budget**. Naive `random()` payouts have two failure modes: overspending (variance) and perceived unfairness (the "I never win" users, or one user winning 5 times).

Design:
1. **Issue** the card on trigger (order > ₹200): create `scratch_card(id, user, state=ISSUED, expires_at)`. **No outcome is decided yet** (or it is decided server-side only, never leaked to the client).
2. **Scratch** → server decides outcome at reveal time (so unspent budget is not pre-committed to unscratched cards):
   - Pull from a **prize table**: `{₹0 or "better luck": 70%, ₹5: 20%, ₹25: 7%, ₹100: 2.5%, ₹1000: 0.5%}` per campaign.
   - **Budget-aware sampling**: expected cost per card is known (`E = Σ p_i * v_i`). A **budget controller** tracks `spent / budget` vs `elapsed / window`. If ahead of the burn curve, shift probability mass to lower tiers; if behind, allow richer prizes. This is a PID-ish or token-bucket pacing loop.
   - **Hard guard**: each prize tier has a Redis token pool (`prize:c_9:₹1000` = 50 remaining); `DECR` and if < 0, fall to the next tier. The total never exceeds the budget even if the probabilities drift.
3. **Fairness**:
   - **Pity timer / bad-luck protection**: guarantee at least a small reward after K consecutive zeros (also a retention tool).
   - **Per-user cap**: max one prize ≥ ₹100 per campaign per user.
   - **Seeded, auditable RNG**: derive outcome from `HMAC(server_secret, card_id)` so the result is deterministic per card (replay-safe, can't be rerolled by retrying `scratch`) and reproducible for dispute review.
   - Publish odds where regulation requires it (loot-box rules; gaming and gambling laws in India differ by state: involve legal for anything that smells like a paid lottery).
4. **Idempotent scratch**: `UPDATE scratch_card SET state = 'REVEALED', outcome = ? WHERE id = ? AND state = 'ISSUED'`; the winner is credited via the normal idempotent earn path with `source_event = card_id`.

> [!NOTE]
> Deciding at **scratch time** with the hard token pool is what makes budget control exact. Pre-assigned prizes ("card 12345 wins ₹100") allow strict fairness but strand budget in unscratched cards; mixed designs pre-generate a **shuffled prize deck per cohort** (sampling without replacement), which gives exact distribution over each cohort of N cards with zero variance.

### Budget Control and Liability Accounting

Two different things:

**Campaign budget (marketing spend, enforced in real time)**:
- `budget_total` per campaign; a Redis counter `budget:c_9:remaining` updated with `DECRBY` in Lua (atomic with per-user cap check).
- Pattern: **reserve then confirm**. Budget Controller reserves `N` before the ledger write; ledger commit confirms; failure returns it. On crash a reconciler compares Redis to `SUM(ledger)` and corrects.
- Hot counter problem: IPL campaign = one key taking 20k ops/sec. Use **sharded budget slices**: split 10M into 100 slices of 100k; each consumer node leases a slice (batch of N tokens), and returns unused tokens on shutdown. Overshoot is bounded by `slice_size x nodes`.
- **Soft vs hard limits**: at 80% alert owner; at 100% auto-pause; per-campaign `overshoot_tolerance` (e.g. 0.5%) is explicit in finance policy.

**Liability (balance-sheet obligation, enforced by accounting)**:
- Outstanding liability = `SUM(user points) x expected redemption value x (1 - breakage_rate)`. Finance books it as deferred revenue (earn on purchase) or marketing expense (promo credit).
- Daily snapshot: per campaign `{issued, redeemed, expired, reversed, outstanding}` written to a warehouse table; month-end journal exported to ERP (SAP/NetSuite) by cost center.
- Breakage (expired / never redeemed) is **income**, which auditors scrutinise; keep clean expiry ledger entries (another reason for batch expiry).

**Reconciliation**:
1. Ledger integrity: `SUM(delta) per txn = 0`; `balance == SUM(entries)` per user (sampled continuously, full weekly).
2. Event completeness: count of source events (orders paid) vs distinct `source_event` in ledger per day; gap = missed events (replay from Kafka/S3).
3. Cash settlement: cashback paid out in the wallet / bank (see [Payment System](./payment-system.md)) vs the ledger's cash account; partner invoices vs `redeemed:pool`.
4. Budget: `campaign.budget_spent == -SUM(ledger campaign expense)`.

### Fraud and Abuse

Rewards are free money, so abuse is guaranteed: multi-accounting, self-referral, emulator farms, wash trading orders and refunding, cashback arbitrage.

Layers:
- **Velocity limits** (cheap, first): earn events per user/device/IP/card per minute/hour/day via Redis sliding windows; per-campaign per-device caps.
- **Device and identity graph**: link accounts by device ID, advertising ID, phone, UPI VPA, card BIN+last4, bank account, IP/ASN, address, contact-book overlap; **connected components** with > K accounts and a shared payout instrument are flagged (graph in Neo4j / Neptune / Flink-built union-find).
- **Referral abuse**: reward on **referee's first successful, non-refunded transaction of ≥ ₹X after a cooling-off (e.g. 7 days)**, not on install; same device / same payment instrument as referrer is blocked; cap referrals per referrer per month; staged payout (50% now, 50% after referee's 2nd order).
- **Pending state**: new accounts' rewards land as `PENDING` (entries tagged, not spendable) until risk window closes (refund window, KYC pass); this converts clawback problems into non-events.
- **ML scoring**: features like account age, order-refund ratio, time between signup and first reward-triggering action, session emulator signals; output `ALLOW | HOLD | DENY`. Run sync for high-value grants, async (post-hoc clawback) for the rest.
- **Wash patterns**: buy-then-refund loops, merchant collusion (self-owned merchant), UPI circular transfers to farm cashback; detect via cycle detection on payer-payee graph; require *real merchant settlement* before reward.
- **Operational**: manual review queue, bulk clawback tool that goes through the **same reversal path** (ledger-correct, audited), allow/deny lists.

Fraud checks must not sit in the critical path of redemption latency; but **never skip them on large redemptions / withdrawals to bank**.

### Tiers and Streaks

**Tiers** (Silver / Gold / Platinum):
- Qualification metric over a rolling window: `qualifying_points` or spend in last 12 months, **separate from spendable balance** (spending points must not drop your tier, so keep a distinct `tier_progress` counter fed by earn events, not by the balance).
- Evaluation: stream job (Flink / Kafka Streams) maintains per-user rolling sums; upgrade is immediate, downgrade at period end with a grace period.
- Benefits are rule inputs: `user.tier` in the DSL yields multipliers, higher caps, early access. Tier changes emit `tier.changed` for notifications and cache invalidation.

**Streaks**:
- State: `(user, streak_type, current_len, last_day, best_len)` in the user shard, updated in the same transaction as the daily check-in grant.
- "Day" is in the user's timezone; define the cut-off once and store `last_day` as a date. Freeze tokens ("streak shields") are a ledgered inventory item.
- Milestone rewards (day 7, 30) are earns with `source_event = streak:{user}:{type}:{milestone}:{cycle}` so a re-processed day cannot pay twice.
- Users in a long streak are the most valuable and the most upset by bugs: a **grace and support-override** path is essential.

### Hot Users, Campaign Spikes (IPL, Diwali)

| Hot spot | Mitigation |
|----------|-----------|
| 10x events at match end | Kafka absorbs; consumers autoscale on lag; earn SLA is "eventual" (seconds to minutes); **redemption stays synchronous** and isolated |
| Single campaign budget key | Lease slices to consumers; local token buckets; reconcile asynchronously |
| Single campaign cap counter per user | Counter keyed per `(user, campaign)`, so no global hot key; batch increments |
| Balance read storm after announcement ("check your rewards!") | Redis projection + CDN-friendly short TTL; **stale-while-revalidate** |

Isolation: separate consumer groups and DB connection pools for **earn**, **redeem**, and **batch jobs** so a bulk campaign cannot starve checkout. Load-shed earn first (it can wait), never redeem. Pre-scale before known events and run a **game day** replaying last year's peak.

### Exactly-Once Pipeline: Outbox and Kafka

```
Producer side (Order/Payment service):
   BEGIN; update order; INSERT outbox(topic='orders.paid', key=user_id, payload); COMMIT;
   relay (Debezium CDC or poller) ──► Kafka  (at-least-once, ordered per key)

Consumer side (Earn service):
   at-least-once consume ──► idempotent write (unique idem key) ──► commit offset

Ledger side:
   state change + outbox('ledger.earned') in one DB transaction ──► Kafka ──► projections, notifications
```

- **Key by `user_id`** so one user's events are ordered (earn before reversal) and one consumer owns a shard of users.
- **No dual writes**: never "write DB then publish to Kafka" in two steps; always outbox.
- **DLQ and replay**: poison events go to a DLQ with reason; fixed rules can **replay** from Kafka/S3 by `source_event`, safe because of idempotency.
- Kafka transactions (EOS) only cover Kafka-to-Kafka; for DB sinks the **idempotent consumer** is the dependable mechanism.
- See [Idempotent REST APIs](../reliability/idempotency.md).

### Notifications

Trigger from `ledger.*` events (so a notification means "really in the ledger"): earned (batched), scratch card available, expiring soon (T-30/T-7/T-1 via daily bucket scan), redeemed, reversal explanation, tier change. Dedup per `(user, template, entity)`; honour quiet hours. See [Notification System](./notification-system.md).

### GDPR / DPDP, Audit and Compliance

- **Audit**: every entry has actor (system / admin id), reason code, rule version, source event, and timestamp; admin adjustments need maker-checker approval and are ledgered like any other movement.
- **Right to erase**: ledger is a financial record (retention obligations override erasure). Pattern: keep ledger rows keyed by a **pseudonymous `user_ref`**; on erasure delete the mapping `user_ref -> PII` (crypto-shredding with per-user keys), leaving anonymised financial entries.
- **Consent and marketing**: reward notifications are marketing communications; honour opt-out per channel.
- **Regulatory**: stored-value / wallet rules (RBI PPI for cash-convertible rewards), gaming/lottery laws for probabilistic rewards, tax (TDS on prizes over a threshold in India), expiry disclosure and breakage accounting. Keep a **program terms version** on every grant.
- **Access control**: support staff see balances and ledger but not raw PII; large adjustments need two-person approval.

### Multi-Region

- **User-homed regions**: each user belongs to a home region/shard; their ledger and buckets live there with a **single writer** (strong consistency for hold/commit).
- Cross-region consumers: events are produced in the region where they occur and routed (by `user_id` hash) to the home region via Kafka MirrorMaker / Cluster Linking.
- **Reads**: balance projection replicated read-only to other regions; redemption always routes to the home region (extra RTT is acceptable; correctness over latency).
- **Budget** is global: allocate **regional slices** of each campaign's budget; rebalance periodically; unspent slice returns to the global pool.

### Storage Choices

| Data | Store | Why |
|------|-------|-----|
| Ledger entries, buckets, holds, idempotency | Postgres / MySQL (sharded by `user_id`) or Spanner/CockroachDB | ACID, constraints, per-user single-writer |
| Balance projection (read) | Redis (write-through from ledger events) + DB row | Sub-ms reads; rebuildable |
| Campaign rules and config | Postgres + in-memory cache (pub/sub invalidation) | Versioned, small, read-mostly |
| Budget / cap / velocity counters | Redis (Lua atomic ops) with periodic DB reconciliation | Hot, high-throughput, approximate-then-exact |
| Event bus | Kafka (keyed by `user_id`) | Ordering per user, replay, lag-based autoscale |
| Device / identity graph | Graph DB or Flink state + feature store | Connected-component and neighbourhood queries |
| Analytics, campaign ROI, finance reports | ClickHouse / BigQuery / Snowflake | OLAP fed from ledger events |
| Cold ledger archive | S3 Parquet (partitioned by month) | Cheap, audit, forensic queries |

### Failure Modes

| Failure | Handling |
|---------|---------|
| Consumer crashes mid-event | Redelivery; unique idem key makes the retry a no-op |
| Duplicate upstream event with different id | Compound business key (`order_id + type`) as `source_event`; fraud/reconciliation counts catch the rest |
| Refund arrives before earn | Tombstone / pending-reversal record; apply on arrival |
| Redis budget counter lost | Rebuild from `SUM(ledger)` per campaign; pause campaign for the rebuild window or fail closed to a conservative lease |
| Ledger shard down | Redemption on that shard fails closed (checkout continues without points); earn events queue in Kafka and drain on recovery |
| Rule misconfiguration pays 100x | Per-txn cap + per-campaign budget + anomaly alert (spend rate vs forecast) + kill switch + bulk reversal tool |
| Projection drifts from ledger | Drift detector; rebuild from entries; serve ledger-derived balance on mismatch |

### Observability

- **SLIs**: earn lag (event time to ledger time) p50/p95/p99, redeem success rate and latency, hold-to-commit conversion, DLQ depth, consumer lag per partition.
- **Integrity**: unbalanced txns (must be 0), projection-vs-ledger mismatches, negative balances count, held-points age.
- **Business**: points issued / redeemed / expired / reversed per day, campaign burn rate vs plan, cost per incremental order, cap-hit rate, fraud deny rate, scratch-card payout vs expected value (drift alarms if variance > 3 sigma).

### Common Mistakes

| Mistake | Reality |
|---------|---------|
| Storing only a `balance` column | No audit, no explainability, no expiry, no clawback; keep ledger plus buckets |
| Random UUID as idempotency key per publish | Retries create duplicates; use stable business keys |
| Single running balance with a single expiry | Cannot expire per earn; use buckets |
| Checking cap in app, then writing ledger separately | Race lets users exceed caps; make check + write atomic or reserve-confirm |
| Pre-deciding scratch outcomes without a budget guard | Variance overspends; use token pools / shuffled decks |
| Evaluating campaign window by processing time | Lag at deadline = wrong eligibility; use event time with grace |
| Tier based on current balance | Spending drops your tier; track qualifying points separately |

> [!NOTE]
> Three non-negotiables: **idempotency keyed by the upstream business event**, **immutable double-entry ledger with balance as a rebuildable projection**, and **budget/caps enforced atomically with the grant**. Everything else (rules DSL, scratch cards, tiers) is layered on those.

### Interview Follow-ups

- *"How do you guarantee a user is never credited twice for one order?"* — Stable `source_event` (e.g. `order_id:payment_success`) plus a unique `(source_event, rule_id, kind)` constraint inserted in the same transaction as the ledger entries; Kafka redelivery becomes a no-op.
- *"User redeemed points, then refunds the order that earned them. What now?"* — Reversal consumes the original buckets if possible; otherwise balance goes negative (debt) via the reversal path only, future earns pay it down, and redemption is blocked meanwhile. For cash-converted points, raise a receivable.

