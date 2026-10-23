# Q: Design an inventory management system.

**Answer:**

An inventory system is the **source of truth for "how many can I sell and where is it."** It looks like CRUD on a counter but is really a **concurrency, ledger, and reconciliation** problem: many channels (web, app, stores, marketplaces) decrement the same stock while physical reality (receipts, damages, theft, miscounts) drifts away from the database.

The interview tests: **on-hand vs reserved vs available**, **atomic decrement without overselling**, **append-only stock ledger (event sourcing)**, **reservations with TTL**, **hot SKUs**, **idempotency**, and **reconciliation with the physical world**.

### Requirements

**Functional**:
- Track stock per `(SKU, warehouse/location, batch)`.
- Receive stock (inbound/PO/GRN), putaway, pick, pack, ship, return, transfer between warehouses, adjust (damage/loss).
- Reserve stock for carts/orders; release on cancel/expiry; commit on shipment.
- Available-to-promise (ATP) queries: "can I sell 3 of SKU X to pincode Y?"
- Batch/lot and **expiry tracking** (FEFO for perishables/pharma).
- Barcode / QR / RFID scanning at receiving, picking, counting.
- Reorder points and purchase suggestions; demand forecasting.
- Cycle counts and reconciliation; full audit trail.
- Sync stock levels to multiple sales channels.

**Non-functional**:
- **Never oversell** (or oversell only within a deliberate, bounded policy).
- Reservation latency < 50 ms p99; ATP read < 20 ms p99.
- Strongly consistent stock mutations per `(SKU, location)`; eventually consistent aggregated views.
- Complete, immutable audit trail (who/what/when/why) for every unit moved.
- Scale to flash sales (thousands of decrements/sec on one SKU) and millions of SKUs.
- High availability; degrade safely (fail closed on reserve, never fail open).

**Out of scope**: procurement/vendor negotiation, WMS robotics, transport routing.

### Capacity Estimation

- 5M SKUs × 200 locations (warehouses, dark stores, shops) = up to **1B stock rows** (sparse: most SKUs live in a few locations; realistically ~100–300M active rows).
- Row size ~200 B → 100M rows ≈ 20 GB hot data: **fits in memory-backed or SSD-backed store comfortably.**
- Ledger entries: 20M orders/day × 3 lines × ~4 movements (reserve, commit, pick, ship) = **~240M entries/day** (~2.8k/sec avg, ~30k/sec peak) ≈ 50 GB/day × 200 B → partition by time, archive to S3.
- Peak flash sale: 50k reservations/sec across the system; **one hot SKU may take 5–10k/sec**.
- ATP reads: 10x writes → ~300k/sec; served from cache/read-model.
- Scanner events: 5k devices × 1 scan/2 s = 2.5k/sec.

Writes concentrate on a handful of hot rows; reads are fan-out heavy and cacheable.

### Core Concepts: On-hand, Reserved, Available

```
on_hand        physical units in the building (per last confirmed movement)
reserved       units promised to orders but not yet shipped (soft allocation)
available      on_hand - reserved - safety_stock          (what you may sell)
in_transit     units shipped to us / between locations (not yet sellable)
damaged/hold   quarantined, QC hold, blocked units       (not sellable)
```

```
ATP(sku, location) = on_hand - reserved - blocked - safety_stock
                     + expected_inbound (optional, for backorder-friendly products)
```

- **Safety stock** buffers forecast error and sync lag.
- **Reservation** lowers `available` immediately but leaves `on_hand` unchanged until the pick/ship event.
- **Commit** converts a reservation into a real deduction at ship time (or pick time).
- Keep these as separate counters; derive `available` rather than storing it where possible (or store it as a *generated* column kept in sync by the same atomic statement).

### API Design

```
POST /v1/inventory/reservations
     Idempotency-Key: <uuid>
     { order_ref, ttl_seconds, lines:[{sku, qty, location_hint?}] }
     → 201 { reservation_id, lines:[{sku, location, qty}], expires_at }
     → 409 { insufficient: [{sku, requested, available}] }

POST /v1/inventory/reservations/{id}/commit      (idempotent)
POST /v1/inventory/reservations/{id}/release     (idempotent)
POST /v1/inventory/reservations/{id}/extend      { ttl_seconds }

GET  /v1/inventory/availability?sku=..&location=..           # ATP (cached)
GET  /v1/inventory/availability/batch?skus=..&pincode=..

POST /v1/inventory/movements        # receive, putaway, pick, adjust, transfer
     Idempotency-Key: <uuid>
     { type:"RECEIPT", sku, location, batch_no, qty, expiry, ref:"PO-123", actor }
GET  /v1/inventory/ledger?sku=..&location=..&from=..&to=..   # audit trail
POST /v1/inventory/cycle-counts     { location, lines:[{sku, bin, counted_qty}] }
```

### High-Level Architecture

```
Sales channels (Web / App / POS / Marketplaces)      Warehouse (WMS, scanners)      Procurement / ERP
          │ reserve / ATP                                   │ movements                  │ POs / receipts
          ▼                                                 ▼                            ▼
   ┌────────────────────────── Inventory API (idempotent) ───────────────────────────────┐
   │  Reservation Service        Movement/Ledger Service        ATP Query Service         │
   └──────┬───────────────────────────┬───────────────────────────────┬──────────────────┘
          │ atomic conditional updates │ append-only                    │ reads
          ▼                            ▼                                ▼
   Stock Balance Store         Stock Ledger (event log)         Redis / read model
   (per sku,location,batch)    (immutable, partitioned)          (availability cache)
          │                            │
          └──────────── outbox ───────►Kafka "inventory-events"
                                         │
        ┌──────────────┬────────────────┼─────────────────┬──────────────────┐
        ▼              ▼                ▼                 ▼                  ▼
  Channel sync    Reorder / Replen   Forecasting     Reconciliation     Analytics / Audit
  (marketplaces)  (reorder points)   (ML)            (cycle counts)     (S3 / ClickHouse)
```

Two views of the same truth:
- **Ledger** (what happened): immutable stock movements. Source of truth for audit and recomputation.
- **Balance** (what is now): current counters per key, updated atomically with each ledger append. Enables O(1) reads and conditional reserves.

### Data Model

```sql
-- Current balances: one row per (sku, location, batch). Hot, small, indexed.
CREATE TABLE stock_balance (
    sku          TEXT        NOT NULL,
    location_id  INT         NOT NULL,
    batch_no     TEXT        NOT NULL DEFAULT '-',
    expiry_date  DATE,
    on_hand      INT         NOT NULL CHECK (on_hand  >= 0),
    reserved     INT         NOT NULL DEFAULT 0 CHECK (reserved >= 0),
    blocked      INT         NOT NULL DEFAULT 0 CHECK (blocked  >= 0),
    version      BIGINT      NOT NULL DEFAULT 0,
    updated_at   TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (sku, location_id, batch_no),
    CHECK (reserved + blocked <= on_hand)
);
CREATE INDEX ON stock_balance (location_id, sku, expiry_date);   -- FEFO picking

-- Immutable ledger: never UPDATE or DELETE. Corrections = compensating movements.
CREATE TABLE stock_ledger (
    id             BIGSERIAL PRIMARY KEY,
    movement_id    UUID        NOT NULL,         -- idempotency / groups multi-leg moves
    sku            TEXT        NOT NULL,
    location_id    INT         NOT NULL,
    batch_no       TEXT        NOT NULL DEFAULT '-',
    movement_type  TEXT        NOT NULL,         -- RECEIPT, RESERVE, RELEASE, COMMIT, PICK, SHIP,
                                                 -- RETURN, ADJUST, TRANSFER_OUT, TRANSFER_IN, DAMAGE, COUNT_ADJ
    delta_on_hand  INT         NOT NULL DEFAULT 0,
    delta_reserved INT         NOT NULL DEFAULT 0,
    reason_code    TEXT,
    ref_type       TEXT,                         -- ORDER, PO, TRANSFER, COUNT
    ref_id         TEXT,
    actor          TEXT        NOT NULL,         -- user, scanner, system
    occurred_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (movement_id, sku, location_id, batch_no)       -- idempotency
) PARTITION BY RANGE (occurred_at);

CREATE TABLE reservation (
    id            UUID PRIMARY KEY,
    order_ref     TEXT NOT NULL,
    status        TEXT NOT NULL,                 -- HELD, COMMITTED, RELEASED, EXPIRED
    expires_at    TIMESTAMPTZ NOT NULL,
    created_at    TIMESTAMPTZ DEFAULT now()
);
CREATE TABLE reservation_line (
    reservation_id UUID, sku TEXT, location_id INT, batch_no TEXT, qty INT,
    PRIMARY KEY (reservation_id, sku, location_id, batch_no)
);
CREATE INDEX ON reservation (status, expires_at) WHERE status = 'HELD';   -- expiry sweeper

CREATE TABLE idempotency_key (
    key TEXT PRIMARY KEY, request_hash TEXT, response JSONB, created_at TIMESTAMPTZ DEFAULT now()
);

CREATE TABLE reorder_policy (
    sku TEXT, location_id INT, reorder_point INT, reorder_qty INT, safety_stock INT,
    lead_time_days INT, PRIMARY KEY (sku, location_id)
);
```

Invariant: `stock_balance` for any key equals the **sum of ledger deltas** for that key. A nightly job asserts this; mismatches page an engineer.

### Atomic Decrement — The Core Problem

Wrong (race condition):

```
qty = SELECT available FROM stock WHERE sku=X;   -- A reads 1, B reads 1
if qty >= 1: UPDATE stock SET available = qty - 1; -- both succeed → oversold
```

Correct options, in order of preference:

**1. Single conditional UPDATE (the default answer).**

```sql
UPDATE stock_balance
   SET reserved = reserved + :qty, version = version + 1, updated_at = now()
 WHERE sku = :sku AND location_id = :loc AND batch_no = :batch
   AND on_hand - reserved - blocked >= :qty;
-- affected rows = 1 → success; 0 → insufficient
```

The database evaluates the predicate and applies the change atomically under the row lock. No read-modify-write in app code. Pair with a ledger `INSERT` in the **same transaction**.

**2. Pessimistic row lock (`SELECT ... FOR UPDATE`).** Correct and simple for multi-row logic (multi-line reservations, FEFO across batches) but serialises contending transactions and risks deadlocks → **always lock rows in a canonical order** (sorted by `(sku, location, batch)`).

**3. Optimistic locking (version column).**

```sql
UPDATE stock_balance SET reserved = :new_reserved, version = :v + 1
 WHERE sku=:sku AND location_id=:loc AND version = :v;   -- retry on 0 rows
```

Fine under low contention; collapses under hot SKUs (retry storms). Prefer conditional update for counters.

**4. Redis + Lua (hot path gate).**

```lua
-- KEYS[1] = "avail:{sku}:{loc}", ARGV[1] = qty
local a = tonumber(redis.call('GET', KEYS[1]) or '0')
if a >= tonumber(ARGV[1]) then
  redis.call('DECRBY', KEYS[1], ARGV[1]); return 1
else return 0 end
```

Lua scripts are atomic in Redis; sub-millisecond, 100k+ ops/sec per node. Use as an **admission gate** in front of the durable store (see Hot SKUs), not as the sole source of truth (failover can lose acknowledged writes).

**5. DynamoDB conditional write / transaction.** `UpdateItem ... ConditionExpression "on_hand - reserved >= :q"`; `TransactWriteItems` for multi-line orders (up to 100 items, all-or-nothing).

**6. Serialise via a log.** Partition by SKU into Kafka; a single consumer per partition applies commands in order (no locks, trivially correct, throughput bound by one thread per SKU).

> [!NOTE]
> Rule: **check and mutate in one atomic operation owned by the datastore.** Any "read in the app, then write" design is an oversell bug waiting for a flash sale.

### Reservation Lifecycle with TTL

```
 reserve(order, lines, ttl)           payment ok / picked / shipped          cancel / failure / timeout
 ────────────────────────► HELD ───────────────────────────────► COMMITTED
                             │
                             ├── release()  ───────────────────► RELEASED
                             └── expires_at passes ────────────► EXPIRED (auto-release)
```

- **Reserve**: for each line (sorted), conditional UPDATE `reserved += qty` + ledger `RESERVE` + reservation rows — one transaction. If any line fails, the transaction rolls back (all-or-nothing for the order) or return a partial result (product decision).
- **TTL**: `expires_at = now + ttl` (checkout 10–15 min; cart-hold in flash sales 2–5 min; B2B orders: no TTL, explicit release).
- **Expiry sweeper**: query `status='HELD' AND expires_at < now()`, release each (idempotent, `UPDATE reservation SET status='EXPIRED' WHERE id=? AND status='HELD'`; only if that affected a row, decrement `reserved`). Use `FOR UPDATE SKIP LOCKED` so multiple sweepers cooperate.
- Alternatively a delay queue (Redis ZSET / SQS delay) triggers release at `expires_at`; the sweeper is the safety net.
- **Commit**: `reserved -= qty; on_hand -= qty` + ledger `COMMIT/SHIP`; idempotent via reservation status CAS (`HELD → COMMITTED`).
- **Late commit after expiry**: re-reserve if stock remains; otherwise fail and trigger payment void / backorder / apology flow.
- **Extend**: payment redirects (3DS) call `extend` so we don't expire mid-payment.

### Event Sourcing the Stock Ledger

- Every change is an **immutable event** (`ReceiptRecorded`, `StockReserved`, `StockPicked`, `StockAdjusted`...). Current balance = fold over events.
- Benefits: perfect audit trail; time travel ("what was on hand at 3pm Tuesday?"); rebuild balances after a bug; feed multiple projections (channel sync, forecasting, finance valuation).
- Cost: complexity, storage, read-side projections needed.

**Pragmatic hybrid (recommended)**: transactional **balance table + append-only ledger written in the same DB transaction**. You get O(1) conditional updates *and* a full audit log without needing a pure event-store. A pure event-sourced approach (balance only derived) makes the atomic "check available ≥ qty" harder because you must keep an in-memory aggregate per `(sku, location)` loaded from snapshot + events and serialise commands per aggregate.

**Snapshots**: every N events (or daily) persist balance snapshots so rebuilds don't replay from the beginning.

**Corrections**: never edit history. A miscount creates a `COUNT_ADJ` ledger entry with reason and approver.

**Outbox**: ledger/balance change and its `inventory-events` outbox row commit atomically; Debezium/poller publishes to Kafka; consumers dedupe by `movement_id`.

### Multi-Warehouse Stock & Allocation

Order lines must be allocated to a node:

```
allocate(order):
  candidates = locations with ATP >= qty for each line, serviceable to pincode
  score(candidate) = shipping_cost + delivery_days_penalty + split_shipment_penalty
                   + stock_balancing_term (prefer overstocked nodes) + capacity_penalty
  pick best single node that fulfils all lines; else minimal-split plan
  then reserve at chosen location(s)
```

- **Virtual / network ATP**: a read model aggregating availability across nodes for catalog display (sum or max by region); the **reserve** step is always local to a node.
- **Transfers** between warehouses: `TRANSFER_OUT` at source (reduces on_hand, adds `in_transit`), `TRANSFER_IN` at destination on receipt. Two ledger legs under one `movement_id`; in-transit is its own state so units are never "lost" while in a truck.
- **Rebalancing**: forecasting suggests moving stock from overstocked to understocked nodes.
- **Ownership of a row** is single-writer by location/shard — a location's stock mutations route to its shard, avoiding distributed transactions for single-node orders. Multi-node orders become a **saga** (reserve per node; compensate on failure).

### Batch / Lot Tracking and Expiry (FEFO)

Perishables and pharma require traceability (recall by batch) and **First-Expired-First-Out**:

```sql
-- choose batches for a pick, oldest expiry first, skipping expired and blocked
SELECT batch_no, on_hand - reserved - blocked AS avail
  FROM stock_balance
 WHERE sku = :sku AND location_id = :loc
   AND (expiry_date IS NULL OR expiry_date > current_date + :min_shelf_life_days)
   AND on_hand - reserved - blocked > 0
 ORDER BY expiry_date NULLS LAST
 FOR UPDATE SKIP LOCKED;
```

- Allocate across batches until the requested qty is satisfied; reserve per batch row.
- **Minimum remaining shelf life** per channel/customer (e.g., retailers demand ≥ 70% of shelf-life remaining).
- **Expiry job** moves expired stock to `blocked`/write-off with a ledger entry and alerts; near-expiry stock triggers markdown/liquidation workflows.
- **Recall**: query ledger by `batch_no` to find every shipment containing that batch and notify customers.
- FIFO vs FEFO vs LIFO: costing method (finance) can differ from picking method (operations); keep separate.
- Serial-numbered items (electronics): per-unit table `item(serial, sku, status, location, order_id)` instead of counters.

### Barcode / Scanning Workflows

```
Receive:  scan PO + scan item barcode + enter qty/batch/expiry → RECEIPT movement → putaway task
Putaway:  scan bin barcode + item → TRANSFER (staging → bin)    (bin-level tracking)
Pick:     pick list per order/wave → scan bin, scan item → PICK movement; mismatch = block + alert
Pack:     scan items into box; verify against order; generate label
Count:    scan bin + item, enter counted qty → compare vs system → variance report
```

- Barcodes: GTIN/EAN/UPC (product), GS1-128 (batch/expiry), internal bin labels; RFID for bulk reads in retail.
- Scanner apps work **offline-first**: queue movements locally with a client-generated `movement_id`; sync idempotently.
- Pick-by-scan reduces mis-picks, the main source of phantom inventory.
- Movement APIs are **idempotent**: scanners double-fire and networks retry.
- Optionally track `bin_id` as part of the balance key: `(sku, location, bin, batch)`; ATP aggregates across bins.

### Reconciliation & Cycle Counts

Systems drift: theft, damage, mis-picks, receiving errors, unscanned movements.

- **Cycle counting** instead of annual full counts: count A-class (high-value/high-velocity) SKUs weekly, B monthly, C quarterly; or count a bin when the system hits zero but it still looks full ("zero-stock verification") or on a pick failure.
- Workflow: freeze movements for the bin (or snapshot expected qty at count time and account for in-flight movements) → count → variance = `counted − expected` → within tolerance auto-adjust; above tolerance requires approval → write `COUNT_ADJ` ledger entry with reason.
- **Pick-short feedback**: if the picker can't find the item, immediately decrement `on_hand` (or block the bin) so we stop selling it, and open a count task.
- **System-vs-system reconciliation**: ERP/WMS vs inventory service vs channel listings vs 3PL feeds — nightly diff job, alert on > threshold.
- **Internal consistency check**: `balance == SUM(ledger)` per key; `reserved == SUM(active reservation lines)`; auto-repair `reserved` drift by recomputing from reservations (it's derived state).
- Track **inventory accuracy %** (record accuracy) and **shrinkage %** as KPIs.

### Hot SKUs & Flash Sales

One row, thousands of concurrent decrements → row lock convoy.

1. **Redis admission gate**: preload `avail:{sku}` = stock available for the sale. Lua `DECRBY` per request; losers get instant "sold out" (protects the DB from 99% of traffic). Winners proceed to the durable conditional UPDATE. If the DB step fails, `INCRBY` to give the unit back. Periodically reconcile Redis to DB.
2. **Sharded counters (bucketing)**: split 1000 units into N=16 sub-rows `(sku, shard_id)`; pick a random shard; fall back to others; reads sum shards. Reduces contention ~N×; adds complexity at the "last few units" tail (visit multiple shards).
3. **Single-writer per SKU**: queue partitioned by SKU; serial application removes contention entirely and gives fair FIFO ordering. Throughput = 1 thread's rate (tens of thousands/sec in memory), fine for most hot SKUs.
4. **Batch/group commit**: aggregate many reservations arriving within 5–10 ms into one DB update (`reserved += sum`), then split results — reduces row-lock operations dramatically.
5. **Limit per-customer quantity** and use a **waiting room** so demand is shaped before inventory.
6. **Allocate stock pools per channel/region** (e.g., 20% for app) to isolate contention and failure.
7. Keep the **ledger insert** off the contended row (it's append-only, naturally partitionable).

### Idempotency

Every mutating call carries an `Idempotency-Key` (or deterministic `movement_id`):

- `reservations`: key = checkout attempt id. Replay returns the same reservation.
- `commit/release`: naturally idempotent via status CAS (`WHERE status='HELD'`).
- `movements` from scanners/ERP: unique `(movement_id, sku, location, batch)` in the ledger; duplicate insert fails with a constraint violation → treat as success.
- Insert idempotency record **in the same transaction** as the effect; if the transaction commits, the effect and the key exist together. See [Idempotent REST APIs](../reliability/idempotency.md).
- Event consumers (channel sync, projections) dedupe by event id since Kafka is at-least-once.

### Concurrency Control — Choosing

| Technique | Use when | Watch out |
|-----------|----------|-----------|
| Conditional `UPDATE ... WHERE avail >= qty` | Default for counters | Need multi-row ordering for multi-line orders |
| `SELECT FOR UPDATE` (+ `SKIP LOCKED`) | Multi-row logic (FEFO across batches) | Deadlocks → canonical lock order; keep tx short |
| Optimistic version column | Low contention (admin edits, config) | Retry storms on hot rows |
| Redis Lua | Hot SKU gate, sub-ms path | Durability on failover; reconcile |
| Distributed lock (Redlock) | Rarely; cross-system critical sections | Not safe for correctness alone; use fencing tokens |
| Serialised log per SKU | Extreme hot SKUs, strict fairness | Single-threaded throughput per SKU |

Isolation: `READ COMMITTED` plus conditional updates is sufficient; `SERIALIZABLE` adds retries without need if updates are atomic.

### Multi-Channel Synchronisation

Channels: own website, mobile app, stores (POS), Amazon/Flipkart/Shopify marketplaces.

- **Central inventory as single source of truth**; channels get *allocations* (a quota/buffer per channel) or consult central ATP in real time.
- **Push model**: on `inventory-events`, a Channel Sync service computes `publishable_qty = available − channel_buffer` and pushes to each marketplace API (rate-limited, batched, retried, coalesced so only the latest value is sent).
- **Pull/real-time** for owned channels: ATP service directly.
- **Oversell protection across channels**: higher safety stock for slow-to-sync marketplaces; lower the published count as it approaches zero (publish `0` when `< threshold`); reservation at order import time with automatic cancel + apology for the loser channel.
- **Order import** from marketplaces is a reserve call (idempotent by marketplace order id).
- **Last-write-wins is dangerous**: never let a channel overwrite the master quantity with its own number; accept only *deltas/events* or controlled full-sync with reconciliation.

### Forecasting & Replenishment

**Reorder point (ROP)**:

```
ROP = (avg_daily_demand × lead_time_days) + safety_stock
safety_stock = z × σ_daily_demand × √lead_time_days       (z ≈ 1.65 for 95% service level)
order qty (EOQ) = √(2 × annual_demand × ordering_cost / holding_cost)
```

- Trigger: after each movement (or on a schedule) if `available + on_order ≤ ROP` → create a suggested PO (human approval or auto).
- **Forecasting**: time-series per SKU/location (ETS/Prophet/LightGBM, hierarchical reconciliation), features: seasonality, promotions, price, weather, events; separate fast-mover vs intermittent-demand models (Croston/TSB).
- **Min/max** policies for simple stores; **days-of-cover** targets for perishables.
- Feedback: measure forecast error (WAPE), stock-outs (lost sales), overstock (holding cost); tune safety stock.
- Inbound visibility: expected receipts (ASN) feed `expected_inbound` into ATP for backorderable items ("ships in 3 days").

### Audit Trail & Compliance

- Ledger is append-only: DB permissions deny UPDATE/DELETE; periodic hash-chain/Merkle checkpoint or WORM storage (S3 Object Lock) for tamper evidence.
- Every entry has `actor`, `reason_code`, `ref_id`; manual adjustments require approval workflow and are flagged for review.
- Query patterns: by SKU timeline, by order, by batch (recall), by actor (shrinkage investigation).
- Retention: hot partitions 90 days in the DB, then Parquet in S3 (Athena/Trino) for 7+ years (financial/tax rules).
- Inventory valuation (FIFO/weighted average) computed from ledger for accounting.

### Storage Choices

| Data | Store | Why |
|------|-------|-----|
| Stock balances | Postgres/MySQL (sharded by sku or location) or DynamoDB | Row-level atomic conditional updates, transactions |
| Stock ledger | Postgres partitioned by time → Kafka → S3/Parquet | Immutable, append-only, audit |
| Reservations | Same DB as balances (same transaction) | Atomic reserve + record |
| Hot SKU gate / ATP cache | Redis (Lua) | Sub-ms atomic counters, reads |
| Events | Kafka | Channel sync, projections, forecasting |
| ATP / network view | Redis + materialised read model (ClickHouse/Elasticsearch) | Fast, eventually consistent |
| Barcode/bin master data | Postgres | Reference data |
| Forecast features/models | Data lake + feature store | ML |
| Archive/analytics | S3 + Athena/Trino | Cheap, queryable history |

Sharding: by `sku` (even spread, hot keys isolated) or by `location` (locality for multi-line single-warehouse orders, simpler transfers). A common compromise: shard by `location_id` for transactional mutations; maintain `sku`-keyed read models for cross-location views.

### Failure Modes

| Failure | Handling |
|---------|---------|
| Reserve succeeded but caller crashed/timeouts | TTL expiry auto-releases; idempotent retry returns same reservation |
| Commit called twice | Status CAS (`HELD→COMMITTED`) makes second a no-op |
| Redis gate lost on failover (counter too high) | DB conditional update still authoritative → no oversell; reconcile counter from DB |
| Redis counter too low (leaked decrements) | Reconcile job rebuilds from DB; expire gate keys per sale window |
| Partial multi-line reservation | Single transaction (all-or-nothing) or compensating releases in saga |
| Deadlock between concurrent multi-line reserves | Lock in sorted `(sku, loc, batch)` order; retry on 40P01 |
| Duplicate scanner submissions | Unique `movement_id` in ledger |
| Kafka outage | Outbox buffers; channel sync lags; widen safety stock; DB remains correct |
| Channel API down/rate-limited | Retry with backoff, coalesce updates, mark channel stale, lower published qty |
| Physical ≠ system (phantom stock) | Pick-short feedback, cycle counts, auto-block bin, reconcile |
| Balance drift vs ledger | Nightly invariant check; rebuild from ledger; page on mismatch |
| Sweeper down → stuck reservations | Multiple sweepers with `SKIP LOCKED`; alert on oldest HELD age; availability queries ignore reservations past `expires_at` |
| DB shard overload | Per-shard rate limiting, move hot SKU to dedicated shard/gate, shed non-critical writes |
| Region outage | Locations homed to a region; failover with replicated ledger; reservations idempotent on retry |

### Observability

- Metrics: reservation success/failure, p99 latency, conflict/retry rate, reserved-but-expired %, oldest HELD reservation age, ATP cache hit rate, outbox lag, channel sync lag, inventory accuracy %, shrinkage, stock-outs.
- Alerts: negative balance (should be impossible → CHECK constraint catches), `balance != SUM(ledger)`, `reserved != SUM(reservation lines)`, oversell detected (orders > stock), sudden zero availability for many SKUs (bad feed).
- Tracing by `order_ref` / `reservation_id`; ledger query is the forensic tool.

### Common Mistakes

| Mistake | Fix |
|---------|-----|
| `SELECT qty` then `UPDATE qty` in app code | Single conditional UPDATE / Lua / Dynamo condition |
| One `quantity` column | Separate on_hand, reserved, blocked; derive available |
| Mutable balance with no history | Append-only ledger in same transaction |
| Reservations with no expiry | TTL + sweeper + explicit release |
| Non-idempotent decrement APIs | Idempotency keys + unique movement ids |
| Treating cart add as a reservation | Reserve at checkout; cart only shows availability hints |
| Letting channels overwrite master stock | Central source of truth; deltas/events; per-channel buffers |
| Ignoring physical drift | Cycle counts, pick-short feedback, reconciliation KPIs |
| Hot SKU on one DB row, no strategy | Redis gate / sharded counters / single-writer queue |
| Locking multi-line orders in arbitrary order | Canonical lock ordering |
| FIFO picking for perishables | FEFO with minimum shelf-life rules |
| Float quantities for discrete items | Integers (use decimal only for weight/volume items with units) |

> [!NOTE]
> Think of inventory as a **bank account for physical goods**: a ledger of movements, a balance derived and guarded atomically, holds (reservations) that expire, and regular reconciliation against reality. The code is easy; the discipline (idempotency, immutability, reconciliation) is the interview.

### Low-Level Design Appendix (OOD Round)

**Requirements**: manage products and stock across warehouses; add/remove stock; reserve and release; low-stock alerts; thread-safe operations; stock movement history.

**Classes**:
- `Product(sku, name, category, reorderPoint)`
- `Warehouse(id, name, location, Map<String, StockItem> stock)`
- `StockItem(product, onHand, reserved)` with `available() = onHand - reserved`
- `Reservation(id, orderId, lines, expiresAt, status)`
- `StockMovement(id, sku, warehouseId, type, delta, reason, timestamp)` (immutable ledger entry)
- `InventoryService` (facade): `addStock`, `reserve`, `commit`, `release`, `getAvailability`, `adjust`
- `enum MovementType { RECEIPT, RESERVE, RELEASE, COMMIT, ADJUST, TRANSFER }`
- `ReorderAlertListener` (Observer) notified when `available < reorderPoint`

**Patterns**: **Observer** for low-stock alerts; **Strategy** for allocation (`NearestWarehouse`, `LowestCost`) and picking (`FIFO`, `FEFO`); **Command** for movements (undo = compensating command); **Facade** for `InventoryService`.

```java
public final class StockItem {
    private final String sku;
    private int onHand;
    private int reserved;
    private final ReentrantLock lock = new ReentrantLock();

    public StockItem(String sku, int onHand) { this.sku = sku; this.onHand = onHand; }

    public int available() { return onHand - reserved; }

    /** Atomic check-and-reserve: the in-memory analogue of the conditional UPDATE. */
    public boolean tryReserve(int qty) {
        lock.lock();
        try {
            if (available() < qty) return false;
            reserved += qty;
            return true;
        } finally { lock.unlock(); }
    }

    public void release(int qty) {
        lock.lock();
        try { reserved = Math.max(0, reserved - qty); } finally { lock.unlock(); }
    }

    public void commit(int qty) {            // shipment: physical stock leaves
        lock.lock();
        try {
            if (reserved < qty || onHand < qty) throw new IllegalStateException("inconsistent");
            reserved -= qty; onHand -= qty;
        } finally { lock.unlock(); }
    }

    public void receive(int qty) {
        lock.lock();
        try { onHand += qty; } finally { lock.unlock(); }
    }
}

public record StockMovement(UUID id, String sku, String warehouseId, MovementType type,
                            int delta, String reason, Instant at) {}

public interface LowStockListener { void onLowStock(String sku, String warehouseId, int available); }

public class Warehouse {
    private final String id;
    private final ConcurrentMap<String, StockItem> stock = new ConcurrentHashMap<>();
    private final List<StockMovement> ledger = new CopyOnWriteArrayList<>();   // append-only
    private final List<LowStockListener> listeners = new CopyOnWriteArrayList<>();
    private final Map<String, Integer> reorderPoints = new ConcurrentHashMap<>();

    public Warehouse(String id) { this.id = id; }

    public void receive(String sku, int qty, String ref) {
        stock.computeIfAbsent(sku, s -> new StockItem(s, 0)).receive(qty);
        ledger.add(new StockMovement(UUID.randomUUID(), sku, id, MovementType.RECEIPT, qty, ref, Instant.now()));
    }

    public boolean reserve(String sku, int qty, String orderId) {
        StockItem item = stock.get(sku);
        if (item == null || !item.tryReserve(qty)) return false;
        ledger.add(new StockMovement(UUID.randomUUID(), sku, id, MovementType.RESERVE, qty, orderId, Instant.now()));
        int avail = item.available();
        if (avail <= reorderPoints.getOrDefault(sku, 0))
            listeners.forEach(l -> l.onLowStock(sku, id, avail));
        return true;
    }
    // release / commit mirror reserve and append their own ledger entries
}

public class InventoryService {
    private final List<Warehouse> warehouses;
    private final AllocationStrategy strategy;          // Strategy: nearest, cheapest, balanced
    private final Map<String, Reservation> reservations = new ConcurrentHashMap<>();

    /** Idempotent by orderId: a retried call returns the same reservation. */
    public Reservation reserve(String orderId, Map<String, Integer> lines, Duration ttl) {
        return reservations.computeIfAbsent(orderId, id -> {
            List<Allocation> done = new ArrayList<>();
            for (var e : new TreeMap<>(lines).entrySet()) {          // sorted order avoids deadlocks
                Warehouse w = strategy.choose(warehouses, e.getKey(), e.getValue());
                if (w == null || !w.reserve(e.getKey(), e.getValue(), orderId)) {
                    done.forEach(a -> a.warehouse().release(a.sku(), a.qty(), orderId));   // compensate
                    throw new InsufficientStockException(e.getKey());
                }
                done.add(new Allocation(w, e.getKey(), e.getValue()));
            }
            return new Reservation(id, done, Instant.now().plus(ttl));
        });
    }
}
```

LLD follow-ups: *thread safety* (per-item lock vs `AtomicInteger` CAS on a packed state; avoid one global lock), *expiry* (a `ScheduledExecutorService` / `DelayQueue` of reservations), *undo* (compensating commands, not mutation of history), *new allocation rule* (new Strategy), *multi-warehouse transfer* (two movements under one id; in-transit state), *serial vs batch tracking* (subclass or composition: `BatchStockItem` holding a `TreeMap<expiry, qty>` for FEFO).

### Interview Follow-ups

- *"How do you prevent overselling when 10,000 people buy the last 100 units?"* — Atomic conditional decrement owned by the datastore (`UPDATE ... WHERE on_hand - reserved >= qty`), plus a Redis Lua admission gate for hot SKUs so only ~100 requests reach the durable store; losers fail fast with "sold out."
- *"Why separate on-hand, reserved and available?"* — Reserved units are promised but still physically present; conflating them loses the ability to expire reservations, cancel orders, or reconcile with physical counts. Available is derived.
- *"Why an append-only ledger instead of just updating a quantity?"* — Audit, debugging, recompute-after-bug, time travel, reconciliation, and feeding downstream systems. Balance is a cache of the ledger's sum; corrections are compensating entries.
- *"Is full event sourcing necessary?"* — Not usually. A balance table plus ledger in one transaction gives atomic conditional updates and an audit log. Full event sourcing needs per-aggregate serialisation and snapshots; worth it when you need strict temporal queries and many projections.
- *"What if the reservation service crashes after reserving but before replying?"* — Client retries with the same idempotency key and gets the same reservation; if the client never returns, TTL expiry frees the stock.
- *"How do you pick the reservation TTL?"* — Based on the funnel stage: checkout 10–15 min, flash sale 2–5 min, B2B no TTL. Extend during payment redirects. Shorter TTL = more stock returned quickly but more "reservation expired" failures.
- *"Optimistic or pessimistic locking?"* — For counters, neither: use a single conditional UPDATE. Pessimistic (`FOR UPDATE`) for multi-row FEFO allocation with ordered locks; optimistic where contention is low. Optimistic on hot rows causes retry storms.
- *"How do you avoid deadlocks on multi-item orders?"* — Lock/update rows in a canonical sorted order, keep transactions short, retry on deadlock, or reserve per line in a saga with compensation.
- *"One SKU is on fire — how do you scale it?"* — Redis gate, sharded counters, single-writer queue per SKU, group commit, per-channel pools, waiting room and purchase limits; keep ledger appends off the hot row.
- *"What are the downsides of sharded counters?"* — Complexity at the tail (last units spread across shards), summing for reads, rebalancing shards when one empties. Mitigate with fallback probing and a periodic rebalance.
- *"Redis says sold out but the DB has stock (or the reverse) — which is right?"* — The DB (or ledger) is authoritative. Redis is a gate; if it's too optimistic the DB conditional update rejects; if too pessimistic a reconcile job corrects it; expire gate keys per sale window.
- *"How is FEFO implemented?"* — Batch rows with expiry; allocate ordered by `expiry_date ASC` with `FOR UPDATE SKIP LOCKED`, exclude batches below minimum shelf life, split a reservation across batches.
- *"How would a product recall work?"* — Query the ledger by `batch_no` to find all outbound movements and orders, block remaining stock, notify customers; this is the traceability argument for batch-level ledger entries.
- *"How do you handle returns?"* — `RETURN` movement into a `quarantine/blocked` state; after inspection, move to sellable stock, write-off, or refurbish; refund is independent of restock.
- *"How do you keep multiple channels consistent?"* — Central inventory is truth; channels get computed publishable quantities (available minus buffer) pushed by event with coalescing; order import reserves idempotently; safety stock for slow channels.
- *"A marketplace order arrives for stock we no longer have — what then?"* — Reserve fails → cancel/backorder and notify the marketplace; track cancel rate, widen buffers, publish 0 sooner.
- *"How do you detect and correct drift between system and shelf?"* — ABC cycle counts, pick-short feedback, zero-stock verification, nightly system-to-system diffs, ledger-vs-balance invariants; adjustments via approved ledger entries.
- *"What do you do when a count reveals a variance?"* — Compare counted vs expected adjusted for in-flight movements; within tolerance auto-adjust, above tolerance require approval and investigation; record reason codes to find systemic causes.
- *"How do you make scanner updates safe offline?"* — Client-generated movement ids queued locally and replayed idempotently; server resolves ordering via `occurred_at` plus validation; conflicts flagged for review.
- *"How do you transfer stock between warehouses without losing units?"* — Two-leg ledger entries under one id with an explicit in-transit state; receiving confirms; discrepancy at receipt creates an adjustment/claim.
- *"How do you compute available-to-promise across warehouses for the PDP?"* — Aggregated read model (Redis/ES) built from `inventory-events`, filtered by serviceable locations for the pincode; eventually consistent; authoritative reserve at checkout.
- *"Should reserve write synchronously to the DB and Redis?"* — DB is the commit point; Redis updated as a derived cache/gate. If using Redis as the first gate, apply the DB step second and roll back the gate on failure.
- *"How do you shard the inventory DB?"* — By `location_id` (keeps a node's operations local and transfers simple) or by `sku` (spreads hot SKUs); maintain the other as a read model. Cross-shard orders use saga reservations per node.
- *"How do you guarantee events are published when stock changes?"* — Transactional outbox in the same DB transaction; relay to Kafka; consumers dedupe by `movement_id`; periodic reconciliation compares channel quantities to master.
- *"How would you compute reorder points and quantities?"* — `ROP = demand × lead time + safety stock`, safety stock from demand variability and desired service level; EOQ for order size; forecast-based dynamic ROP; consider MOQ, pack sizes, supplier lead-time variance.
- *"How do you forecast intermittent-demand SKUs?"* — Croston/TSB or probabilistic models rather than standard seasonal models; aggregate to higher hierarchy and reconcile; use service-level targets per class.
- *"Which isolation level?"* — `READ COMMITTED` with atomic conditional updates; avoid `SERIALIZABLE` unless multi-statement invariants can't be expressed atomically; always add `CHECK` constraints as a last line of defence.
- *"What if the DB constraint `on_hand >= 0` fails in production?"* — That means a code path decremented without the guard; the constraint prevents corruption and the failing transaction rolls back; alert, investigate via ledger, and fix the path.
- *"How do you test for races?"* — Concurrency tests firing N parallel reserves for M < N units and asserting exactly M successes; chaos tests killing nodes mid-transaction; property tests that `balance == SUM(ledger)`.
- *"How do you handle multi-tenant (many sellers) inventory?"* — Include `tenant_id` in the key/partition; per-tenant quotas and rate limits; noisy-neighbour isolation for hot tenants; separate ledgers or row-level security.
- *"How do you keep read latency low for availability checks?"* — Cache ATP in Redis with short TTL, update by events, batch endpoints, return coarse buckets ("in stock", "low stock", exact only below a threshold), serve stale on cache failure.
- *"What about eventual consistency between ATP display and the reserve step?"* — Acceptable by design; the reserve is authoritative. Quantify the rate of "displayed in stock but reserve failed" and tune via event latency and safety stock.
- *"How do you migrate from a single warehouse to many?"* — Introduce `location_id` into keys with a default location, backfill ledger, dual-write, then enable allocation strategies; keep API backward compatible via location hints.
- *"How do you secure and audit manual adjustments?"* — RBAC, approval workflows for large adjustments, reason codes, immutable ledger with actor, anomaly alerts on adjustment patterns per user, periodic review.
- *"How do you retain history cheaply?"* — Time-partition the ledger, detach old partitions to Parquet on S3, keep aggregates (daily balances) in the hot DB; Athena/Trino for forensic queries.
- *"What would you monitor and alert on?"* — Reservation success and latency, retry/conflict rate, oldest HELD age, ledger-balance mismatch, negative or impossible state, outbox lag, channel sync lag, inventory accuracy %, stock-outs vs forecast.
- *"Evolution path?"* — Single Postgres with conditional updates and a ledger → add Redis gate and outbox/Kafka → shard by location, add channel sync and forecasting → network-wide ATP, probabilistic allocation, automated replenishment and rebalancing.
