# Q: Design a car rental / car booking system (Zoomcar, Turo, Hertz).

**Answer:**

A reservation system for **time-bounded exclusive resources**. Unlike seat booking (a point in time), a car is booked for an **interval** (pickup to return), so the central problem is **interval-overlap correctness** plus availability search over date ranges, pricing that varies by duration and demand, money held as deposits, and a physical handover lifecycle (pickup, return, damage). *Note: on-demand cab hailing (Uber/Ola) is a different problem (real-time geo matching), covered in the ride-sharing chapter.*

### Requirements

**Functional**:
- Search cars by location, date/time range, vehicle class/make, price, features (automatic, SUV, EV).
- Show availability and total price (rental + taxes + fees + deposit) for a date range.
- Reserve a car (or a class), pay or pre-authorize, receive confirmation.
- Pickup (digital key / counter / delivery) and return, with check-in/check-out inspection.
- Extend, modify, cancel with policy-based refunds.
- Deposits, insurance add-ons, fuel/mileage charges, late fees, damage claims.
- Host side (Turo/Zoomcar marketplace): list cars, set availability calendar, pricing, approve requests.

**Non-functional**:
- **No double-booking** of a specific vehicle for overlapping intervals.
- Search p99 < 500 ms with accurate-enough availability; booking strongly consistent.
- Booking volume is modest but each booking is high-value (money, long intervals); correctness > raw throughput.
- Handle timezone, DST, and one-way rentals across locations.
- Auditable money trail (deposits, captures, refunds).

### Capacity Estimation

```
Fleet: 500k vehicles globally, 5k branches/cities
Bookings: 2M/day  -> ~25/s avg, ~150/s peak (holiday spikes 5x)
Searches: 100:1 search-to-book -> 200M/day -> ~2.3k/s avg, ~15k/s peak
Availability data: each car has ~10 future bookings -> 5M interval rows (tiny)
Telematics (connected cars): 500k vehicles x 1 ping/10 s = 50k writes/s (time-series, separate store)
```

Writes are low; reads (search over date ranges) dominate. The hard part is *correctness of intervals* and *searching availability efficiently*, not throughput.

### Core Concepts: Two Inventory Models

| Model | Example | What the customer reserves | Pros | Cons |
|-------|---------|---------------------------|------|------|
| **Specific vehicle** | Turo, Zoomcar | This exact car (plate X) | Simple correctness, exact photo/host | Cancellations of a specific car hurt; less flexible |
| **Vehicle class (pooled)** | Hertz, Avis, Enterprise | "Compact or similar" at a branch | High utilisation, easy re-assignment, overbooking possible | Needs allocation step at pickup, upgrade logic |

Hybrid is common: reserve a **class** (counter-based inventory) and **assign a specific vehicle** late (at pickup or T-24h). Marketplace products reserve specific vehicles.

**Class-based inventory** is a counting problem per (branch, class, time): at any instant, `booked_count <= fleet_count`. This is **interval-overlap counting**, harder than exclusive-resource locking (you cannot use a simple unique constraint).

### API Design

```
GET  /v1/search?lat=&lng=&radius=&start=&end=&class=&transmission=&sort=
       -> [{ listing_id, vehicle_class, price_total, price_per_day, distance, host_rating, ... }]
GET  /v1/listings/{id}/availability?from=&to=         -> calendar (free/busy ranges)
POST /v1/quotes   { listing_id|class+branch, start, end, addons[], driver_id }
       -> { quote_id, price_breakdown, deposit, expires_at }       (price locked 15 min)
POST /v1/reservations
       headers: Idempotency-Key
       body: { quote_id, driver_id, payment_method_id }
       -> 201 { reservation_id, status: "CONFIRMED" | "PENDING_APPROVAL" }
POST /v1/reservations/{id}/modify     { new_start?, new_end?, new_class? }
POST /v1/reservations/{id}/cancel
POST /v1/reservations/{id}/pickup     { checkin_photos[], odometer, fuel, signature }
POST /v1/reservations/{id}/return     { checkout_photos[], odometer, fuel }
POST /v1/reservations/{id}/damage     { photos[], description }       (post-return claim)
POST /v1/webhooks/payments                                              (PSP events)
```

Quote-then-reserve ensures the price shown is the price charged, and lets pricing be computed once, and verified at booking.

### Data Model

```sql
CREATE EXTENSION IF NOT EXISTS btree_gist;

CREATE TABLE vehicle (
    vehicle_id   BIGINT PRIMARY KEY,
    owner_id     BIGINT,                    -- host (marketplace) or fleet company
    branch_id    BIGINT,                    -- home location
    class        TEXT NOT NULL,             -- ECONOMY, SUV, LUXURY
    make_model   TEXT, plate TEXT UNIQUE, vin TEXT UNIQUE,
    status       TEXT NOT NULL,             -- ACTIVE, MAINTENANCE, RETIRED
    geo          GEOGRAPHY(Point)           -- for peer-to-peer cars
);

-- One row per rental, hold, or maintenance/blocked window
CREATE TABLE vehicle_booking (
    booking_id    BIGINT PRIMARY KEY,
    vehicle_id    BIGINT NOT NULL REFERENCES vehicle,
    period        TSTZRANGE NOT NULL,       -- [pickup, return + buffer)
    kind          TEXT NOT NULL,            -- RENTAL, HOLD, MAINTENANCE, HOST_BLOCK
    reservation_id BIGINT,
    hold_expires_at TIMESTAMPTZ,
    -- THE invariant: no two overlapping periods for one vehicle
    EXCLUDE USING gist (vehicle_id WITH =, period WITH &&)
);

CREATE TABLE reservation (
    reservation_id BIGINT PRIMARY KEY,
    renter_id      BIGINT NOT NULL,
    vehicle_id     BIGINT,                  -- null until assigned (class booking)
    branch_id      BIGINT, class TEXT,
    pickup_at      TIMESTAMPTZ NOT NULL,
    return_at      TIMESTAMPTZ NOT NULL,
    status         TEXT NOT NULL,           -- see state machine
    quote_id       BIGINT, total_cents INT, deposit_cents INT,
    cancellation_policy TEXT,
    idem_key       TEXT, UNIQUE (renter_id, idem_key),
    version        INT NOT NULL DEFAULT 0
);

CREATE TABLE inspection (
    inspection_id BIGINT PRIMARY KEY, reservation_id BIGINT,
    phase TEXT,                              -- PICKUP | RETURN
    odometer INT, fuel_pct INT, photos JSONB, notes TEXT, signed_by TEXT, at TIMESTAMPTZ
);

CREATE TABLE payment_ledger (               -- append-only double entry
    entry_id BIGINT PRIMARY KEY, reservation_id BIGINT,
    type TEXT,                               -- AUTH, CAPTURE, DEPOSIT_HOLD, DEPOSIT_RELEASE, REFUND, FEE, DAMAGE_CHARGE
    amount_cents INT, currency TEXT, psp_ref TEXT UNIQUE, created_at TIMESTAMPTZ
);
```

### Overlap Check: The Heart of the Problem

Two intervals `[s1, e1)` and `[s2, e2)` overlap iff `s1 < e2 AND s2 < e1`. Use **half-open intervals** so back-to-back rentals (return at 10:00, next pickup at 10:00) do not conflict. Add **buffer time** (cleaning, inspection) into the blocked period: `[pickup, return + buffer)`.

**Option 1: Postgres exclusion constraint (best when the DB supports it).**

```sql
EXCLUDE USING gist (vehicle_id WITH =, period WITH &&)
```

The database itself rejects an overlapping insert atomically under concurrency (the GiST index checks conflicts including uncommitted concurrent inserts). Booking is simply:

```sql
INSERT INTO vehicle_booking(booking_id, vehicle_id, period, kind, reservation_id)
VALUES (:id, :v, tstzrange(:start, :end + :buffer, '[)'), 'RENTAL', :r);
-- SQLSTATE 23P01 (exclusion_violation) -> "car no longer available"
```

No application locking, no race, strongest guarantee. Downside: Postgres-specific.

**Option 2: Row lock on the vehicle + overlap query (portable, MySQL).**

```sql
BEGIN;
SELECT * FROM vehicle WHERE vehicle_id = :v FOR UPDATE;        -- serialize bookings per car
SELECT 1 FROM vehicle_booking
 WHERE vehicle_id=:v AND start_ts < :end AND end_ts > :start;   -- any overlap?
-- none -> INSERT; commit
```

Per-vehicle lock; contention only between people wanting the same car. Works anywhere. Index on `(vehicle_id, start_ts, end_ts)`.

**Option 3: Slot/day-bucket rows with unique key.**

Pre-materialize `(vehicle_id, day)` or `(vehicle_id, hour_slot)` rows; booking inserts one row per slot with `UNIQUE (vehicle_id, slot)`. A multi-slot insert is all-or-nothing in one transaction. Simple and portable but coarse granularity and write amplification (a 30-day rental = 30 rows); good for day-based rentals.

**Option 4: Interval tree / in-memory structure.**

For search acceleration or a single-writer actor per vehicle: a sorted structure of busy intervals per vehicle (interval tree, or sorted list with binary search: check predecessor end <= start and successor start >= end, O(log n)). Not a source of truth, a cache/accelerator, unless all writes for a vehicle are routed to one owner.

**Option 5: Counter-based for class inventory** (pooled): see below.

> [!NOTE]
> The strongest answer: "Make the invariant a database constraint. Postgres `EXCLUDE USING gist` on `(vehicle_id, tstzrange)` guarantees no overlap even under concurrent inserts; elsewhere lock the vehicle row and check overlap in the same transaction."

### Class-Based Availability (Counting)

For "a Compact at Airport branch from Fri 10:00 to Mon 10:00" we need: `max over time of concurrent bookings < fleet size`.

- **Sweep line** over start/end events within the window: count active reservations at each boundary; max must be below capacity (plus overbooking factor).
- **Materialized counters per (branch, class, time bucket)**: `available[bucket] = fleet - reserved`. Reserving decrements every bucket in the range atomically (all-or-nothing):
  ```sql
  UPDATE inventory_bucket SET reserved = reserved + 1
   WHERE branch_id=:b AND class=:c AND bucket BETWEEN :s AND :e
     AND reserved < capacity;     -- must affect (e - s + 1) rows, else rollback
  ```
  Buckets = hour or day. Cancelling decrements back. Efficient for search (read bucket min over range) and consistent via conditional update.
- **Segment tree** with range-add and range-max gives O(log n) check-and-reserve per (branch, class), useful in-memory or for big fleets.
- Late **vehicle assignment**: at T-24h (or pickup), a batch assigns concrete vehicles with a bipartite matching (reservation requirements vs vehicles: class, transmission, child seat, location); handle upgrades when a class is exhausted.

### High-Level Architecture

```
Web / Mobile
     │
 API Gateway (auth, rate limit, idempotency)
     │
 ┌───┴────────┬──────────────┬─────────────┬──────────────┬──────────────┐
 ▼            ▼              ▼             ▼              ▼              ▼
Search     Pricing       Reservation    Payment       Fleet/Ops      Notification
Service    Service       Service        Service       Service        (SMS/email/push)
 │  ▲         │               │             │              │
 ▼  │         ▼               ▼             ▼              ▼
ES / Geo   Rules +       Postgres        PSP        Telematics ingest
+ Redis    ML model      (bookings,      (auth/     (Kafka → TSDB)
availability              exclusion)     capture)         │
 ▲                          │                             ▼
 └── CDC / events ◄── Outbox ► Kafka ──► Analytics, Fraud, Settlement
```

### Availability Search over Date Ranges

Search must answer "which cars are free for [start, end) near here" fast. Two-stage:

1. **Candidate retrieval (approximate)**: filter by geo (geohash/H3/ES `geo_distance`), class, attributes, price. Pull top K candidates (e.g. 500).
2. **Availability filter (fast, mostly correct)**: remove cars with overlapping busy intervals.
   - Per-vehicle busy intervals cached in Redis (sorted set keyed by start, value=end) or a **bitmap by day** (1 bit per day for the next 365 days: `vehicle:{id}:busy`). Free for a range iff `AND` of bits over range is 0 (use `BITCOUNT range`).
   - In Elasticsearch: store a `busy_ranges` nested/`date_range` field, query with `must_not` `intersects` range; or store per-day availability tokens. Updated by CDC from the booking DB (seconds of lag).
   - Pooled classes: read the bucket counters.
3. **Authoritative check at booking**: the exclusion constraint / conditional update. Search may be stale; booking is never wrong. A stale hit just yields "no longer available, here are similar cars."

Ranking: relevance (distance, price, rating, host response, instant-book), conversion predictions, fairness to hosts. Cache popular (city, date-range) results for 30-60 s.

Host calendar: hosts block dates (`HOST_BLOCK` rows in the same table so the same exclusion constraint covers them), set minimum/maximum trip length, advance notice and prep time.

### Pricing

```
price = base_daily(class, location, season)
      x duration_discount (weekly/monthly tiers)
      x demand_multiplier (utilisation of fleet, events, holidays, lead-time)
      + mileage / fuel / delivery / airport fee
      + insurance add-ons
      + taxes (jurisdiction) + platform/service fee
```

- **Time-based**: hourly, daily, weekly; rounding rules (24 h blocks; grace period of 29 minutes before extra day).
- **Dynamic pricing**: utilisation-based (fewer cars left -> higher), lead-time curve, local events, competitor scraping, ML model; hosts on marketplaces may apply smart pricing suggestions.
- **Quote locking**: price stored in `quote` (immutable, signed), valid ~15 minutes; booking uses the quote, not a recalculation.
- **Taxes and currencies**: computed by a tax service; store both amounts and currency; FX at booking time.
- **Extension pricing**: priced at the current rate, not the original; require availability check for the extra interval.
- **One-way rentals**: drop-off fee based on repositioning cost; inventory moves between branches (balance with rebalancing trucks or pricing incentives).

### Deposits and Payment Pre-Authorization

- **Security deposit** (e.g. 5k-20k INR / $200-500): take a **card pre-authorization (hold)** at booking or pickup, not a charge. Auth holds typically last 7 days (card networks vary, up to 30 for some), so for long rentals **re-authorize** periodically.
- **Rental fee**: authorize at booking, capture at pickup (or charge immediately for prepaid fare), final capture at return after extras (fuel, tolls, mileage, late).
- **Flow**:
  ```
  booking   -> auth(rental_total)             [refundable window]
  pickup    -> auth(deposit)  (+ capture rental)
  return    -> compute final charges -> capture(extra) -> release(deposit)
  claim     -> capture part of deposit for damage (within claim window) -> release remainder
  ```
- **PSP idempotency**: every PSP call carries an idempotency key `reservation_id:step:attempt`; ledger entries are append-only and keyed by `psp_ref UNIQUE`.
- **Auth expiry**: scheduler tracks `auth_expires_at`; re-auth failures trigger a customer notification.
- **Debit cards / UPI** (India): no pre-auth on many instruments, so use **UPI mandates / e-mandates**, refundable prepaid deposit, or wallet blocks; card-on-file for post-trip charges.
- **Do not store PANs**: tokens from the PSP vault.

### Reservation State Machine

```
        quote ok               payment auth ok
(none) ─────────► PENDING_PAYMENT ─────────────► CONFIRMED ──────────┐
                     │ timeout/fail                 │  (host approval flow: PENDING_APPROVAL ─► CONFIRMED / DECLINED)
                     ▼                              │ pickup inspection signed
                  EXPIRED/FAILED                    ▼
                                                 ACTIVE (in rental)
                                                    │ return inspection
                       cancel (policy) ◄── CONFIRMED│        ▼
                          │                       RETURNED ─► CLOSED (final settlement, deposit released)
                          ▼                          │
                      CANCELLED ─► REFUNDED          └─► DISPUTED (damage/claim) ─► CLOSED
                  NO_SHOW (not picked up by grace) ─► CANCELLED w/ penalty
```

Guard every transition with `UPDATE ... SET status=:new, version=version+1 WHERE id=:id AND status=:expected`. Late return is not a state; it is a derived flag (`now > return_at`) that triggers fees and alerts.

### Pickup, Return, and Digital Keys

- **Pickup**: ID + driving licence verification (OCR + liveness, DigiLocker/DMV checks), age/experience rules, payment auth for deposit, vehicle inspection (photos, odometer, fuel), signed agreement.
- **Keyless** (Zoomcar/Turo Go): Bluetooth/telematics unlock via a time-boxed signed token (valid only within `[pickup, return]`); the reservation service issues it, the telematics unit validates offline.
- **Delivery/pickup at address**: scheduling, ETA, driver-assignment (reuse dispatch concepts).
- **Return**: inspection, fuel/odometer, parking location (for free-floating), auto-detect via telematics geofence; compute extras (extra km, fuel, tolls via FASTag/e-toll, late fee).
- **Late return**: grace period, then hourly fees; if the next booking is at risk, trigger **swap/upgrade** logic or compensate the next customer.
- **Early return**: policy-defined (usually no refund), may free inventory (re-list).

### Damage, Insurance, and Claims

- **Check-in/check-out photos with timestamps and geotags** are the evidence; diff against the pickup photos; AI-assisted damage detection reduces disputes.
- **Insurance** options: basic (included, high excess), zero-depreciation/CDW, personal accident; marketplaces use a partner insurer per trip (policy issued at pickup, trip-scoped), P2P hosts covered by platform policy.
- **Claim flow**: report within a window (e.g. 24-72 h after return) -> evidence -> assessment -> capture from deposit (partial) -> remainder released; dispute resolution path; for larger damage, charge the renter's card on file and/or file with the insurer.
- **Telematics evidence**: speed, harsh braking, crash detection, geofence violations (leaving country/state) feed into claims and risk.
- **Fraud & trust**: driver scoring, blacklist, repeated damage flags, host verification, VIN checks, GPS tracking for theft recovery; immobilizer for non-payment.
- Immutable audit trail of inspections and charges (legal disputes).

### Cancellation and Modification Policies

| Policy | Typical rule |
|--------|-------------|
| Flexible | Full refund up to 24 h before pickup, then 1 day charged |
| Moderate | Full refund up to 5 days before, 50% later |
| Strict | Refund up to 7 days before; none after |
| Host cancels | Full refund + penalty to host + re-accommodation offer |
| No-show | Charge first day (or full) after grace period |

- Policy is **snapshotted onto the reservation** at booking (rules can change later).
- Refund computed by a policy engine function `refund(policy, now, pickup_at, paid)`; executed as idempotent ledger entries.
- **Modify** = new interval check first (atomically swap period: insert-new/delete-old in one transaction, or `UPDATE` period with the exclusion constraint re-checking), then price difference charged/refunded.
- **Host/vehicle cancels** (accident, breakdown): automatically search substitute (same or better class, nearby) and re-assign; compensate; this is why pooled classes are more resilient.

### Overbooking

Airlines-style: sell more reservations than cars, because of no-shows and early returns.

- Overbooking factor per (branch, class, day) derived from historical no-show/cancel rates and late-return rates, e.g. +5%.
- When actual demand exceeds supply at pickup: **free upgrade** to a higher class, **partner-branch transfer**, or **compensation / rebooking**; cost model: `E[cost of oversell] vs E[revenue of unsold car]`.
- Only for class-based pooled inventory, never for specific-vehicle marketplace listings (the car either exists or not).
- Safeguards: cap overbooking near the pickup time (when no-show probability is known, tighten), VIP/loyalty tiers never bumped.

### Geo Search and Delivery Radius

- Marketplace: listings have a point; index with geohash/H3 in Elasticsearch/PostGIS; query `geo_distance` or bounding box plus filters.
- Branch-based: map user location -> nearest branches (small set, ~hundreds per city), then per-branch availability.
- Airports/stations as popular pickup nodes; "delivery to me" within radius adds fee.
- Map clustering at low zoom (pre-aggregated by H3 cell counts) so the map view stays fast.
- **Timezones**: store timestamps in UTC plus the branch's IANA timezone; interpret user input in branch-local time; handle DST (a 24 h rental across DST is 23 or 25 clock hours; define the rule in policy).

### Concurrency Scenarios

1. **Two renters, same car, overlapping dates** -> exclusion constraint (or vehicle row lock); loser gets "unavailable".
2. **Renter A holds during checkout while renter B books**: use a short **soft hold** (`kind='HOLD'`, `hold_expires_at`) inserted into the same table (also covered by the exclusion constraint); converted to `RENTAL` on payment success; expired holds are ignored by a partial constraint (`WHERE kind <> 'HOLD' OR hold_expires_at > now()` isn't allowed in exclusion predicate with `now()`, so delete expired holds by sweeper and lazily on conflict).
3. **Modify and new booking racing**: both touch the same vehicle/period; same constraint arbitrates.
4. **Payment webhook vs hold expiry**: same pattern as ticketing: lock the reservation row, extend hold when payment starts, compensate with void/refund if the car was lost.
5. **Host blocks dates while a renter books**: same table, same constraint; host block fails if a confirmed rental overlaps (shows a conflict UI).

### Observability and Operations

- Metrics: search-to-book conversion, search latency, availability-staleness rate (stale hits at booking), booking failures by reason, utilisation per class/branch, no-show rate, late-return rate, claim rate, auth failure rate.
- Alerts: reservations `PENDING_PAYMENT` too long, deposits not released after return, vehicles `ACTIVE` past return by more than X hours (potential theft), telematics offline.
- Reconciliation: nightly ledger vs PSP statements; booking table vs availability index (repair drift).
- Ops tooling: manual override with audit, vehicle swap, maintenance blocks that conflict with bookings (move/upgrade affected renters).

### Scaling and Multi-Region

- **Shard by region/city** (or branch): a booking touches one vehicle in one region. Cross-region one-way rentals are rare and handled as a booking in the pickup region with a drop-off reference.
- Search index global or per region; replicas of the catalogue; availability caches per region.
- Booking DB: Postgres per region with replicas for reads; vehicle count per city is small (thousands), so a single primary per region is comfortable.
- Cache availability but verify at write; never read-then-write from cache.
- Peak seasons: pre-scale search tier; throttle low-value traffic (scrapers) with rate limits and caches.

### Abuse and Fraud

| Threat | Defence |
|--------|---------|
| Stolen cards / identity | KYC (licence OCR + liveness), PSP risk scoring, 3DS, deposit holds, velocity rules |
| Vehicle theft | Telematics tracking, geofencing, immobilizer, deposit, insurance, blacklist |
| Fake listings (marketplace) | VIN/registration verification, photo checks, host KYC, listing review |
| Booking-and-cancel to block inventory | Cancellation fees, deposits, rate limits, holds with short TTL |
| Odometer / fuel disputes | Photos + telematics odometer, geotagged check-in/out |
| Scraping prices | Bot detection, rate limits, authenticated pricing |
| Host/renter collusion (fake trips for promos/payouts) | Graph analysis, GPS trip validation, delayed payouts |

### Storage Choices

| Data | Store | Why |
|------|-------|-----|
| Reservations, vehicle_booking, ledger | Postgres (per region) | ACID, exclusion constraint, FK |
| Search / geo / facets | Elasticsearch / OpenSearch (+ PostGIS) | Geo, filters, ranking |
| Availability bitmaps, quote cache | Redis | Fast range checks |
| Telematics / GPS | Kafka -> TimescaleDB / ClickHouse | High-volume time series |
| Inspection photos, agreements | S3 + CDN, signed URLs | Large blobs |
| Events | Kafka (+ outbox) | Decoupled pipelines |
| Pricing features | Feature store / data warehouse | ML pricing |
| Documents (licence images) | Encrypted S3 + KMS | PII, retention policies |

### Failure Modes

| Failure | Handling |
|---------|----------|
| Two overlapping bookings attempted | Exclusion constraint / vehicle lock; loser gets alternatives |
| Search shows stale availability | Authoritative check at booking; graceful "similar cars" fallback |
| Payment auth succeeds, reservation write fails | Idempotent retry; reconciliation job voids orphan auths |
| Webhook lost / duplicated | Reconciliation by polling PSP; idempotent state transitions |
| Customer doesn't return on time | Late fees, alerts, next-booking mitigation (swap), telematics lock-down escalation |
| Vehicle breaks down before pickup | Auto re-assign same class or upgrade, compensation |
| Deposit auth expires on long rental | Scheduled re-auth, notify, fallback to card-on-file |
| Damage dispute | Photo evidence, telematics, claim workflow, partial deposit capture |
| Elasticsearch lag | Booking path unaffected; temporary stale search |
| Region outage | Replica promotion; bookings in flight retried by idempotency key |
| Timezone/DST bug | Store UTC + branch tz; test on DST boundaries |
| Overbooking exceeds fleet at pickup | Upgrade/transfer/compensation playbook |

### Common Mistakes

| Mistake | Fix |
|---------|-----|
| Checking overlap in app code, then inserting | DB exclusion constraint, or lock vehicle row in the same transaction |
| Closed intervals `[s, e]` causing false conflicts | Half-open `[s, e)` plus explicit turnaround buffer |
| Modelling a rental as per-day boolean only | Use ranges (hour-level) or slot rows, else same-day turnovers break |
| Treating specific-vehicle and class inventory as the same | Class = counting problem; vehicle = exclusive resource; hybrid with late assignment |
| Charging the deposit instead of authorizing | Pre-authorize and release; capture only on claim |
| Not re-authorizing long rentals | Track auth expiry; re-auth or card-on-file |
| Recomputing price at booking | Signed quote with expiry |
| Ignoring timezones/DST | UTC + branch timezone, explicit policy |
| Trusting search availability | Authoritative booking-time constraint |
| No compensation path when a car disappears | Substitute search, upgrade, refund, host penalty |
| Mutable payment records | Append-only ledger with unique PSP refs |
| Single global DB lock for bookings | Per-vehicle / per-branch partitioning |

> [!NOTE]
> The interview signal: you recognise this is **interval scheduling with exclusive resources**, you know the overlap predicate (`s1 < e2 AND s2 < e1`), and you enforce it with a **database constraint** rather than application logic, while splitting **fast approximate search** from **slow authoritative booking**.

### Interview Follow-ups

- *"How do you stop double-booking a car?"* — Postgres `EXCLUDE USING gist (vehicle_id WITH =, period WITH &&)` makes overlap impossible atomically; alternatively `SELECT ... FOR UPDATE` on the vehicle row then overlap query in the same transaction.
- *"Why half-open intervals?"* — `[10:00,12:00)` and `[12:00,14:00)` do not overlap; closed intervals create phantom conflicts at boundaries. Add a buffer to model cleaning/inspection time.
- *"What's the overlap condition?"* — `start1 < end2 AND start2 < end1`; negate it for availability.
- *"How do you search availability fast over date ranges?"* — Two phases: geo + attribute filtering in ES, then an availability filter via per-vehicle busy-interval cache or day-bitmaps; verify authoritatively at booking.
- *"Why can't ES be the source of truth?"* — Eventual consistency lag; no cross-document uniqueness/exclusion guarantees; so it serves discovery only.
- *"How would you do an interval tree here?"* — Per-vehicle sorted busy intervals with binary search (predecessor end <= start, successor start >= end) in O(log n); as an in-memory accelerator or in a single-writer actor per vehicle.
- *"Vehicle-specific vs class-based booking?"* — Specific vehicle: exclusive resource, constraint-protected (Turo). Class: counting inventory with overbooking and late vehicle assignment (Hertz). Hybrid reserving class and assigning near pickup.
- *"How do you check class availability across a range?"* — Per-hour/day bucket counters decremented atomically within a transaction (`reserved < capacity`), or a segment tree with range-add/range-max.
- *"How do you assign a vehicle to a class reservation?"* — Batch bipartite matching at T-24h or at pickup, honoring constraints (class, transmission, location, child seat), with upgrades on shortage.
- *"How do you handle a user who is checking out while another books the same car?"* — Soft hold row (`kind=HOLD` with TTL) inserted under the same constraint; converted on payment; expired by sweeper.
- *"How do you handle payment webhook racing hold expiry?"* — Row-lock the reservation, extend hold when payment starts, confirm via guarded transition; if the car was lost, void/refund and re-accommodate.
- *"Auth vs capture for rental fee?"* — Authorize at booking, capture at pickup/return with final extras; voids are free and instant, refunds slow and costly.
- *"How do deposits work?"* — Pre-authorization hold on the card, released after return and inspection; partial capture for damage within a claim window; re-authorize for long rentals since auths expire.
- *"What if the card has no pre-auth capability (debit/UPI)?"* — UPI mandates or e-mandates, refundable prepaid deposit, wallet hold, or card-on-file for post-trip charges; higher deposit or KYC for riskier instruments.
- *"What about a cancellation after the deposit hold?"* — Release the hold (void), refund the rental per policy snapshot, record ledger entries; all idempotent.
- *"How do cancellation policies work technically?"* — Policy snapshot on reservation; pure function computing refund from time-to-pickup; executed as idempotent ledger entries; host-initiated cancels invert the penalty.
- *"How do you modify a booking (extend/shorten/change car)?"* — Re-run availability for the new period atomically (swap period within one transaction; constraint rechecks), reprice at current rates, charge or refund the difference; if unavailable, offer alternatives.
- *"How do you handle late returns when the car is booked next?"* — Predictive alert (telematics/ETA), grace period, fees, swap the next customer to an equivalent car or upgrade, compensate when impossible.
- *"How do you handle a vehicle breakdown before pickup?"* — Substitute search (same or higher class nearby), auto re-assign, notify, compensation; pooled classes make this easy, marketplace needs re-search or refund.
- *"Overbooking: how and why?"* — Because of no-shows and early returns; overbooking factor from historical rates per branch/class/day; mitigation: upgrades, transfers, compensation; tighten near pickup time; never for marketplace specific-vehicle listings.
- *"How do you model one-way rentals?"* — Drop-off location differs; inventory shifts across branches; add drop-off fee reflecting repositioning cost; rebalance via pricing incentives and logistics.
- *"How do you handle timezones and DST?"* — Store UTC plus branch IANA timezone; interpret user input at the branch; define 'day' rules in policy; test around DST transitions.
- *"How do you price dynamically?"* — Utilisation, lead time, events, seasonality, competitor rates, duration discounts; ML model with guardrails (floors/ceilings); quote locked with expiry.
- *"Why quotes?"* — Guarantees price consistency between display and charge, supports auditing, and lets pricing be computed once and verified.
- *"How do you handle damage claims?"* — Geotagged timestamped photos at pickup and return, telematics, AI diff, claim window, partial deposit capture, insurer involvement, dispute workflow.
- *"How is insurance integrated?"* — Included basic cover plus add-ons; trip-scoped policies issued via insurer API at pickup; platform policy for P2P; claim data shared via events.
- *"How do you verify renters?"* — Licence OCR + liveness + government verification APIs, age/experience rules, risk score, blacklist, PSP identity checks.
- *"How do keyless unlocks stay secure?"* — Short-lived signed tokens bound to reservation, device and time window; offline verification on the telematics box; revoke on cancel or return.
- *"Geo search for peer-to-peer cars?"* — ES `geo_distance` or PostGIS with filters; cluster at low zoom via H3 aggregates; delivery radius per host.
- *"How do you shard?"* — By city/region (a booking involves one vehicle in one region); per-vehicle row locks/constraints inside; cross-region only for rare one-way trips.
- *"How do you keep search availability index consistent?"* — CDC/outbox events on booking changes update ES/Redis; periodic reconciliation rebuild; booking-time check as the safety net.
- *"What if the primary DB fails?"* — Failover to replica; clients retry with idempotency keys; in-flight payments reconciled by PSP polling; stale-hit rate temporarily higher.
- *"How do you prevent fake listings or fraud on a marketplace?"* — Host KYC, VIN/registration verification, photo validation, delayed payouts, graph analysis for collusion, telematics GPS trip validation.
- *"How do you prevent inventory blocking by serial bookers?"* — Short holds, cancellation fees, deposit/prepayment, rate limits, risk scoring.
- *"How do you test this?"* — Concurrency tests (parallel overlapping inserts; exactly one winner), property tests on interval logic incl. adjacency/DST, PSP failure simulation, replay of peak-season traffic, invariant checker (no overlapping rows per vehicle).
- *"What do you monitor?"* — Conversion funnel, stale-availability rate, booking failures, utilisation, late-return/no-show rates, pending payments/aging, ledger vs PSP reconciliation, telematics offline.
- *"How would it evolve (subscriptions, EV charging, fleet-owner dashboards)?"* — Subscriptions = recurring reservation with rolling interval; EV adds state-of-charge and charging-buffer constraints on availability; host dashboards from analytics warehouse; dynamic pricing ML iterates on the feature store.
- *"How does this differ from cab hailing?"* — Rental reserves a vehicle for an interval in advance with no live matching; hailing is real-time geo-matching with seconds-scale supply state (see the ride-sharing chapter).
- *"How do you reduce cost?"* — Cache availability and search results, cheap storage for photos (lifecycle to cold), batch assignments, keep ES small by indexing only upcoming windows, shard by region to avoid cross-region traffic.
- *"What's your MVP?"* — Postgres with exclusion constraint, single PSP with auth/capture, specific-vehicle inventory, ES or Postgres geo search, manual inspections; add pooling, overbooking and dynamic pricing later.
