# Q: Design an online food delivery platform (Swiggy / Zomato / DoorDash).

**Answer:**

A **three-sided marketplace** (customer, restaurant, delivery partner) with a hard real-time constraint: food gets cold and customers get angry. The interesting parts are **geo-aware discovery**, the **order saga** across payment + restaurant + dispatch, **delivery partner assignment** (a batching/assignment optimisation problem, not "nearest driver"), and **peak-hour scaling** (lunch, dinner, IPL finals produce 5–10x spikes in minutes).

Compared to ride-sharing, there is a third actor with its own latency (the kitchen) and a prediction problem (food prep time) that dominates the quality of dispatch.

### Requirements

**Functional**:
- Customer browses restaurants near a delivery address; search by dish / cuisine / restaurant name; filters (veg, rating, ETA, offers).
- View menu with live item availability and price.
- Cart, apply coupons, see itemised bill with delivery fee, taxes, surge.
- Place order and pay (UPI / card / wallet / COD).
- Restaurant accepts or rejects, marks preparing, marks ready.
- Delivery partner (DP) assignment, pickup, drop, proof of delivery (OTP).
- Real-time order tracking with ETA.
- Ratings and reviews (restaurant, food, delivery partner).
- Cancellation and refunds; support tickets.
- Push / SMS / in-app notifications at every state change.

**Non-functional**:
- Browse path: p99 < 300 ms, highly available (AP). A stale menu for a few seconds is fine.
- Order placement path: strong consistency on order + payment. Never double-charge, never lose a paid order.
- Dispatch decision within 2–5 seconds of order confirmation.
- Tracking updates every 3–5 s with < 2 s end-to-end lag.
- Survive 10x spikes at meal times and during marquee events.
- Auditability for money, GST invoices, restaurant payouts.

**Out of scope (state explicitly)**: grocery / quick-commerce dark stores (different inventory model, see [Inventory Management](inventory-management.md)), restaurant POS integration internals, fraud ML internals.

### Capacity Estimation

Assume a Swiggy-scale India system:

- 50M MAU, 10M DAU, 2M orders/day at average, 4M on peak days.
- Peak hour carries ~15% of daily orders: 2M × 0.15 = 300k orders/hour = **~85 orders/sec**; with festival/IPL skew, plan for **300 orders/sec**.
- Browse-to-order ratio ~ 50:1 → **~15k browse/search QPS** at peak.
- Delivery partners: 400k active, ~150k online at peak. Location ping every 4 s → **~40k writes/sec** (and ~150k during IPL surges when partners are all active).
- Tracking: each in-flight order has 1 customer polling/subscribed. 100k concurrent in-flight orders × 1 update / 4 s = **25k fan-out msgs/sec**.
- Menu data: 300k restaurants × 100 items × 2 KB = **60 GB** — fits in memory across a cache tier / search index.
- Order record ~5 KB (with items) × 4M/day = 20 GB/day = ~7 TB/year, plus event log.

Like ride-sharing, scale is dominated by **location ingest and read-heavy discovery**, not order volume. Orders are low QPS but money-critical.

### API Design

```
GET  /v1/restaurants?lat=..&lng=..&q=biryani&filters=veg,rating4+&sort=relevance&cursor=
GET  /v1/restaurants/{rid}/menu
POST /v1/cart                       { restaurant_id, items:[{item_id, qty, addons[]}], address_id }
POST /v1/cart/price                 → itemised bill (fees, taxes, discounts, ETA)
POST /v1/orders                     Idempotency-Key: <uuid>
                                    { cart_id, payment_method, address_id, price_token }
GET  /v1/orders/{oid}               → state, timeline, ETA, DP info
GET  /v1/orders/{oid}/track         → WebSocket / SSE upgrade (live location)
POST /v1/orders/{oid}/cancel
POST /v1/orders/{oid}/rating

# Restaurant app
POST /v1/restaurant/orders/{oid}/accept   { prep_time_minutes }
POST /v1/restaurant/orders/{oid}/reject   { reason }
POST /v1/restaurant/orders/{oid}/ready
POST /v1/restaurant/items/{id}/availability { in_stock: false, until: ts }

# Delivery partner app
POST /v1/dp/location                (batched pings)
POST /v1/dp/offers/{offer_id}/accept|reject
POST /v1/dp/orders/{oid}/arrived|picked_up|delivered   { otp }
```

Notes:
- `price_token` is a signed, short-lived (e.g., 5 min) quote from the pricing service; placing the order with a stale token forces a re-quote. This prevents "price changed between cart and pay" disputes.
- `Idempotency-Key` on `POST /orders` — mobile networks retry constantly. See [Idempotent REST APIs](../reliability/idempotency.md).

### High-Level Architecture

```
Customer app ──► API Gateway / BFF ───┬──► Discovery (Search + Ranking) ──► Elasticsearch / geo index
                                      ├──► Menu Service ──► Menu DB + Redis (availability overlay)
                                      ├──► Cart / Pricing Service ◄── Fee/Surge/Coupon services
                                      └──► Order Service (saga orchestrator)
                                                 │  outbox → Kafka
        ┌───────────────┬──────────────┬─────────┴───────┬──────────────┬─────────────┐
        ▼               ▼              ▼                 ▼              ▼             ▼
   Payment Svc    Restaurant Svc   Dispatch Svc      Tracking Svc   Notification   Ledger/Payout
   (PSP + ledger) (accept/timeout) (assign/batch)    (WS fan-out)   (push/SMS)     (settlements)
                       │               │  ▲
                       ▼               ▼  │ DP location (Kafka → Redis geo)
              Restaurant tablet/   DP app ┘
              POS integration           │
                                  Prep-time + ETA ML service (feature store)
```

Key principle: **read path and order path are separate systems.** Discovery is an eventually-consistent, denormalised, cached read model (CQRS). The order path is a transactional workflow.

### Restaurant Discovery & Geo Search

Query: "restaurants that **deliver to my location**, open now, matching 'biryani', ranked well."

Two geo concepts that candidates conflate:
1. **Restaurant location** (static point).
2. **Serviceability** — a restaurant delivers to a *polygon / radius around it*, adjusted by road distance, DP supply, and kitchen load. A restaurant 2 km away across a river may be unserviceable.

**Approach (production-realistic)**:

```
Index time:
  restaurant → serviceability cells (H3 res 8/9 ~ 0.5–0.7 km hexes it can serve)
  document in Elasticsearch: { id, name, cuisines[], dishes[], rating, price_bucket,
                               is_open, served_cells[], city, avg_prep_time, popularity }

Query time:
  user_cell = h3(lat,lng, res=8)
  ES query: filter served_cells = user_cell  AND is_open
            must match text q (name^3, dish^2, cuisine)
            function_score: relevance + popularity + rating + ETA + personalisation
```

Why invert the geo relation (restaurant → cells served) instead of `geo_distance` from the user? A pre-computed cell membership is a cheap term filter; it also lets ops encode *non-circular* serviceability (rivers, gated areas, DP-poor zones) without runtime geometry. Cell serviceability is recomputed by a batch / stream job as supply and kitchen load change.

Alternatives:
- `geo_distance` filter in ES / Postgres PostGIS: simple, fine for < 100k restaurants per city and a single radius.
- Geohash prefix: same idea as H3, worse neighbour geometry.
- Pure Redis GEO (`GEOSEARCH`): fast for "nearest N" but no text relevance — use for *nearest DP*, not for restaurant search.

**Sharding the index**: by city. A query never crosses cities. Large cities (Bangalore, Mumbai) get multiple shards by geo-cell range.

**Caching**: popular `(cell, filters)` result pages cached 30–60 s (CDN/Redis). At lunch the hot working set is small — 5k cells × few filter combos.

**Hello-interview style insight**: the home feed is not a search query, it is a **pre-ranked list per cell** (carousels: "Top picks", "Under 30 min", "Offers") produced by an offline/nearline job and personalised by a light re-ranker at request time.

### Search Ranking

Signals (blend with a learned-to-rank model; start with a hand-weighted score):

```
score = w1 * text_relevance
      + w2 * conversion_rate(restaurant, cell, daypart)
      + w3 * rating_bayesian(rating, n_ratings)
      + w4 * (1 / predicted_delivery_eta)
      + w5 * availability (open, kitchen load, stock of searched dish)
      + w6 * personalisation(user cuisine affinity, reorder history)
      + w7 * promo_boost / sponsored (clearly labelled ads)
      - w8 * cancellation_rate / complaint_rate
```

- Use **Bayesian average** for ratings so a 5.0 with 3 reviews doesn't beat a 4.5 with 3,000.
- **Rank by predicted ETA, not distance.** A close restaurant with a 40 min prep queue loses.
- Cold start: new restaurants get exploration traffic (bandit / boosted slot) for a limited window.
- **Never rank a closed or overloaded restaurant high** — the worst experience is ordering and being rejected.
- Dish-level search ("paneer tikka"): index dishes as nested/child docs; collapse results to restaurants and show the matching dish card.
- Typo tolerance / transliteration (Hinglish): n-gram analyzers and synonym lists. See [Typeahead](typeahead.md).

### Menu Service & Availability

Menu data has two very different change rates:

| Part | Change rate | Store |
|------|-------------|-------|
| Catalogue (names, descriptions, images, base price, add-ons) | Rare (daily/weekly) | Postgres/Mongo (source of truth) → ES + CDN-cached JSON |
| Availability (in-stock flags, sold-out until X) | Constantly (restaurant taps "out of stock") | Redis hash `avail:{rid}` + event stream |
| Price / offers | Hourly (happy hours, dynamic) | Pricing service, merged at read time |

**Overlay pattern**: serve the cached catalogue blob and merge the small availability overlay at the BFF. Updating one item toggle never invalidates a 200 KB menu blob.

Restaurant tablets stop working, staff forget to mark items out of stock → **auto-disable** items with high post-order rejection rate or when order-acceptance latency spikes. Also: restaurant working hours + a **heartbeat** from the tablet; no heartbeat for N min ⇒ restaurant marked temporarily closed in search.

> [!NOTE]
> Order-time validation re-checks availability at the authoritative store. Browse-time data can be stale; **order-time data must not be**.

### Cart & Pricing

- Cart is single-restaurant (usually) and short-lived: store in **Redis** (TTL 24 h) keyed by `user_id`, persisted lazily to DB for abandoned-cart marketing.
- **Pricing service is the single source of truth for the bill**:

```
item_total
+ packaging charge
+ restaurant GST (5%)
+ delivery fee   = base(distance band) + surge(cell, time) + rain_fee + long_distance_fee
+ platform fee
- coupons / membership free delivery
= grand_total
```

- Returns a **signed price_token** capturing every component and the ETA promised. The order stores this snapshot; later config changes never alter a placed order.
- Coupons: validate eligibility, per-user usage limits, global budget caps. Budget decrements are atomic counters (Redis `DECR`, reconcile to DB) — see [Distributed Counter](distributed-counter.md).

### Delivery Fee & Surge

- **Distance band** base fee (road distance, from routing service, not haversine).
- **Surge** is driven by `demand/supply` ratio per cell over a sliding window (orders in last 5 min vs idle DPs within radius), smoothed with hysteresis so fees don't flicker.
- Other inputs: rain, peak daypart, late night.
- Lock the fee into the price token (user sees it before paying). Many platforms prefer **fee levers** and **DP incentives** (per-order bonus, "peak hour guarantee") over raising customer prices, because customer elasticity is high.
- Membership (Swiggy One / Zomato Gold) waives the fee; the platform absorbs it from subscription revenue — model as a *discount line* so ledgers still balance.

### Order State Machine

```
CREATED ─► PAYMENT_PENDING ─► PLACED ─► RESTAURANT_ACCEPTED ─► PREPARING ─► READY
   │             │               │              │                 │          │
   │             ▼               ▼              ▼                 ▼          ▼
   │        PAYMENT_FAILED   REJECTED /     CANCELLED          CANCELLED   DP_ASSIGNED ─► PICKED_UP ─► DELIVERED
   │                         TIMED_OUT                                                         │
   ▼                         (auto refund)                                                     └─► DELIVERY_FAILED
 EXPIRED                                                                                            (customer unreachable)
```

Rules:
- Transitions are validated in the Order Service via a **transition table** (`from, event → to`); illegal transitions return 409.
- Every transition = `UPDATE orders SET status=?, version=version+1 WHERE id=? AND status=? ` (compare-and-set), plus an `order_events` row, plus an **outbox** event in the same transaction.
- Cancellation allowed until `PICKED_UP`; cost of cancellation depends on state (free before accept, partial fee after prep started).

```sql
CREATE TABLE orders (
    id              BIGINT PRIMARY KEY,         -- snowflake id
    customer_id     BIGINT NOT NULL,
    restaurant_id   BIGINT NOT NULL,
    dp_id           BIGINT,
    status          TEXT NOT NULL,
    version         INT NOT NULL DEFAULT 0,
    price_snapshot  JSONB NOT NULL,             -- full bill, fees, taxes, discounts
    address         JSONB NOT NULL,             -- snapshot (not FK)
    promised_eta_at TIMESTAMPTZ,
    placed_at       TIMESTAMPTZ DEFAULT now(),
    delivered_at    TIMESTAMPTZ
);
CREATE TABLE order_items (
    order_id BIGINT, item_id BIGINT, name TEXT, unit_price NUMERIC(10,2),
    qty INT, addons JSONB, PRIMARY KEY (order_id, item_id)
);
CREATE TABLE order_events (
    id BIGSERIAL PRIMARY KEY, order_id BIGINT, from_status TEXT, to_status TEXT,
    actor TEXT, payload JSONB, at TIMESTAMPTZ DEFAULT now()
);
CREATE INDEX ON orders (customer_id, placed_at DESC);
CREATE INDEX ON orders (restaurant_id, status);
```

Orders sharded by `customer_id` (history queries) or `order_id`; restaurant dashboards read from a **separate read model** (restaurant-keyed) fed by events.

### Order Placement Saga

```
1. Order Svc:   validate cart + price_token + availability   → CREATED
2. Payment Svc: authorize / collect (or mark COD)            → PAYMENT_PENDING → PLACED
3. Restaurant:  push order to tablet; wait for ACCEPT (SLA 60–90 s)
                   ├─ ACCEPT  → PREPARING, prep_time known
                   └─ REJECT / TIMEOUT → compensation: cancel + refund
4. Dispatch:    scheduled so DP arrives ≈ when food is READY (not at accept time)
5. DP:          pickup → in-transit → delivered (OTP)
6. Settlement:  capture payment, record ledger entries, restaurant + DP payables
```

**Orchestrated saga** (Order Service as orchestrator) is preferable to choreography here: ~8 steps, many compensations, strict timeouts, and you need one place to answer "where is order X stuck?".

Compensations:

| Step failed | Compensation |
|-------------|--------------|
| Payment failed | Order → PAYMENT_FAILED; release coupon; nothing to refund |
| Restaurant rejected / timed out | Void auth or refund; release coupon; notify; suggest alternatives |
| No DP found (after retries) | Escalate: incentive bump, then cancel + refund + compensation voucher |
| Customer cancels after prep started | Charge cancellation fee; pay restaurant for food cost |
| Delivery failed | Policy-based (customer unreachable → no refund; DP fault → refund) |

Use **auth-then-capture** for cards: hold at placement, capture on pickup/delivery, void on rejection (cheaper than refund; no settlement fees, instant to the customer). For UPI (no auth/hold in practice) you collect and *refund* on failure — refund latency is a UX problem, so surface wallet credits for instant compensation.

See [Payment System](payment-system.md) for the ledger and idempotency details.

### Restaurant Acceptance, Rejection & Timeouts

- Restaurant tablet receives order via **push + persistent WebSocket + loud alarm**; phone-call/IVR escalation after 45 s with no response.
- Timeouts implemented via a **delay queue / scheduler** (e.g., Redis ZSET with `due_ts`, Kafka delayed topic, or SQS delay): on order PLACED schedule `restaurant_timeout(order_id)` at +90 s. The timeout handler is idempotent and does a CAS `PLACED → TIMED_OUT`; if the restaurant already accepted, CAS fails and it no-ops. See [Job Scheduler](job-scheduler.md).
- Rejection reasons feed back: item out of stock → auto-disable item; too busy → temporarily lower serviceability radius / hide restaurant.
- Penalise chronic rejectors in ranking; this is a core quality lever.
- Offer **auto-accept** for reliable restaurants with POS integration.

### Prep-Time Prediction

Prep time is the #1 contributor to ETA error and DP idle time.

Features: restaurant, item mix (count, categories), kitchen backlog (open orders right now), time of day, day of week, historical `ready_at - accepted_at` distribution, restaurant's self-declared time, weather, whether it's a festival.

Model: gradient-boosted trees (quantile regression: p50 for planning, p80 for promising customers). Online features (current backlog) via a feature store (Redis) updated by the order stream; offline features daily.

"Food Ready" signal: restaurant taps READY, but they forget. Augment with **DP-at-restaurant dwell time** and geofence arrival events; train on the DP-confirmed pickup time as ground truth (Swiggy's published approach: treat the restaurant's own marking as a noisy label).

### Delivery Partner Assignment

This is the most-asked deep dive. Naive: "assign the nearest free DP." Production: **batched global assignment**.

**Why not greedy nearest-DP on each order?**
- Greedy locks in a locally-good match that steals a DP better suited for the next order 5 s later.
- Ignores prep time: sending the DP at accept time means 10 minutes idle at the restaurant.
- Ignores multi-order batching (two orders from the same mall / same direction).

**Batched matching (DoorDash / Swiggy style)**:

```
every T seconds (e.g., 5s) per zone:
  orders  = unassigned orders with predicted ready_time
  dps     = online DPs (idle, or finishing a drop near the restaurant)
  build cost matrix:
      cost(dp, order) = w1 * dp_to_restaurant_time
                      + w2 * max(0, dp_arrival - food_ready)   # food waiting
                      + w3 * max(0, food_ready - dp_arrival)   # DP waiting (idle)
                      + w4 * expected_delay_to_customer
                      + w5 * dp_fairness / acceptance_history_penalty
  solve min-cost assignment (Hungarian / min-cost flow / greedy-with-lookahead)
  send offers to chosen DPs; on reject → re-enter pool next cycle
```

- **Order batching/bundling**: group orders with the same restaurant (or nearby) and compatible drop-offs if the detour adds < X minutes for each customer. Evaluate the bundle's total delivery time vs two separate DPs.
- **Dispatch timing**: schedule assignment at `ready_time - dp_travel_time - slack` (a "delayed dispatch"), unless supply is thin — then dispatch early.
- **Candidate retrieval**: Redis `GEOSEARCH` / H3 k-ring on the DP index (latest location + status). Filter by status, vehicle type, current load, shift, distance cap.
- **Offer flow**: push offer with TTL (15–20 s). Accept → CAS `DP idle → ASSIGNED` and `order.dp_id IS NULL → dp_id` (so two DPs can't be assigned and an order can't have two DPs).
- **Fairness/earnings**: avoid starving low-rated or far-away DPs; some platforms implement a "queue" of DPs waiting at a hotspot.
- **Fallbacks**: incentive boost, widen radius, cross-zone DPs, third-party fleet (Dunzo-like), finally cancel with compensation.

Alternatives:
1. **Greedy nearest** — simple; fine as MVP and at low density.
2. **Auction/offer-broadcast** (offer to top-K, first accept wins) — lower latency, noisy for DPs.
3. **Batched optimisation** — best efficiency; requires per-zone solver and reliable ETAs.

Interviewer favourite: *"What's the objective?"* Minimise a weighted sum of customer delivery time, food wait (cold food), DP idle time, and DP cost — with SLA constraints.

### ETA Computation

```
promised_ETA = P(accept_delay) + prep_time(p80) + dp_to_restaurant + handover
             + restaurant_to_customer(road, live traffic) + buffer
```

- Road routing: OSRM / Valhalla / in-house graph with learned speed profiles per hour-of-week; fall back to haversine × detour factor.
- Pre-order ETA (listing page) uses cheaper approximations; checkout ETA recomputed with live state.
- Re-estimate in-flight and push updates; **never silently lengthen** by large amounts without a proactive notification (and a delay-compensation policy).
- Track **ETA error** (promised vs actual) per restaurant and cell as the north-star accuracy metric.

### Real-Time Tracking

```
DP app ──ping(lat,lng,heading,ts,battery)──► Location Gateway ──► Kafka "dp-locations"
                                                                     │
                      ┌──────────────────────────────────────────────┤
                      ▼                         ▼                    ▼
              Redis GEO/H3 index       Tracking Service        S3 / ClickHouse
              (latest + status)        (order_id → DP map)     (history, analytics)
                                              │
                                              ▼
                              WebSocket/SSE gateway → customer app
```

- Ping interval adaptive: 4–5 s while on an order, 15–30 s idle, slower on low battery. Batch several pings per HTTP call to save radio power.
- **Only push to the customer whose order the DP currently holds** (privacy + efficiency). Tracking Service maintains `dp_id → active order(s)` from the order events.
- Customers' screens subscribe via WebSocket (fallback long-poll/FCM silent push). Gateway tier is stateless w.r.t. business logic; connection → pod mapping stored in Redis so any producer can route.
- **Map snapping + interpolation** on the client for smooth movement, so you don't need 1 Hz updates.
- Last-known state served from Redis for reconnecting clients.
- See [Geo Proximity](geo-proximity.md) and [Ride Sharing](ride-sharing.md) for index details.

### Notifications

Event-driven fan-out from Kafka order events → Notification Service (templates, per-user preferences, channel selection: push → SMS fallback, WhatsApp for transactional where allowed).

- Dedupe by `(order_id, event_type)`, since consumers get at-least-once delivery.
- Priority tiers: order_confirmed/ DP-arriving are P0; promo is P3 and is dropped under load.
- Restaurant side: voice/IVR fallback for unacknowledged orders.
- See [Notification System](notification-system.md).

### Ratings & Reviews

- Ratings are collected after delivery for **three entities**: food/restaurant, delivery partner, and (optionally) packaging. Each is a separate score with its own consumer (ranking, DP quality, restaurant dashboard).
- Only verified purchasers may rate (an `order_id` is required): one rating per order (unique index on `order_id`).
- Aggregates computed asynchronously (stream aggregator updates `restaurant_rating_stats` incrementally: count, sum, rolling windows); display with Bayesian smoothing and a **recency weight**.
- Moderation: profanity/abuse filtering, "rating bombing" detection, restaurant dispute workflow.
- Low rating on delivery ⇒ trigger automatic support intervention (refund/credit) — a retention lever.

### Refunds & Cancellations

- Refund policy engine maps `(state at cancel, who cancelled, reason)` → `(refund %, restaurant payout %, DP payout)`.
- Refunds are issued by the Payment Service as a **new ledger transaction** (never mutate old entries) to the original instrument, or instantly to wallet/credits when speed matters.
- Idempotent per `(order_id, refund_reason)`; PSP refund API called asynchronously with retries; webhook closes the loop.
- Item-level issues (missing item, spilled): partial refund via support tooling with abuse guardrails (see below).

### Settlement & Payouts

- Order completion emits `order_delivered` → ledger posts: customer funds → restaurant payable (item total − commission), platform revenue (commission + fees), tax liabilities, DP payable.
- Restaurants paid weekly/T+2; DPs daily or on-demand (instant-payout is a retention lever).
- Daily **reconciliation** between PSP settlement files and our ledger; unmatched entries open tickets.

### Abuse & Fraud

| Abuse | Control |
|-------|---------|
| Refund abuse ("food not delivered") by a customer | Per-user refund-rate scoring, DP GPS + OTP proof, escalate to manual review |
| Coupon abuse / multi-accounting | Device fingerprint, phone/payment-instrument linking, per-user coupon caps, graph analysis |
| DP fakes delivery / GPS spoofing | Geofence checks at drop, OTP handover, mock-location detection, speed anomaly |
| Restaurant inflates menu or fakes orders (collusion) | Compare to PoS prices, velocity checks, anomaly detection |
| COD no-shows | COD limit per user, prepaid-only for risky accounts |
| Fake ratings | Verified-order only, bot detection, weight by account age |

### Peak-Hour & IPL-Spike Scaling

Spikes are **predictable** (meal times) and **semi-predictable** (IPL match breaks, festivals, rain). Handle with:

1. **Pre-scaling** from forecast + schedule (match time table), not just reactive autoscaling (new pods take minutes).
2. **Read path shielding**: CDN + Redis cache for menus and listings; stale-while-revalidate; serve degraded (non-personalised) ranking under pressure.
3. **Load shedding by priority**: protect `place order`, `payment`, `tracking`; shed recommendations, carousels, non-critical analytics, promo notifications.
4. **Queue-based decoupling**: order placement writes DB + outbox, everything else async via Kafka.
5. **Rate limiting & admission control** per user and per restaurant; "virtual waiting room" if the system is saturated. See [Rate Limiter](rate-limiter.md).
6. **Restaurant-side throttling**: a kitchen that accepts 40 orders in 10 min destroys ETAs. Cap concurrent orders per restaurant by capacity (auto-pause when backlog > X) — this is the physical bottleneck, not our servers.
7. **Thundering herd** at 7:00 pm: jittered client retries, request coalescing for popular menus, pre-warmed caches.
8. **DP supply levers**: incentive boosts announced ahead of the event, predictive hotspot maps ("go to Koramangala").
9. Per-city cell-based architecture: each city/region is an independent deployment unit; a bad deploy or overload in Mumbai doesn't take Delhi down.

### Data Partitioning & Storage Choices

| Data | Store | Why |
|------|-------|-----|
| Orders, order events, payments | Postgres/MySQL (sharded by customer/order id) | ACID, CAS transitions |
| Restaurant & menu catalogue | Postgres / MongoDB | Source of truth, flexible schema |
| Search index | Elasticsearch / OpenSearch | Text + filters + geo cell |
| Menu availability, carts, sessions | Redis | Hot, TTL, atomic ops |
| DP latest location + status | Redis GEO / H3 sets | Proximity queries, write heavy |
| Location history | Kafka → S3 (Parquet) / ClickHouse | Cheap archive, analytics, ML |
| Event bus | Kafka | Order events, outbox, fan-out |
| Ratings aggregates | Cassandra / Redis + Postgres | Write-heavy counters |
| Images, invoices | S3 + CDN | Blob |
| Feature store | Redis + offline lake | ETA/prep-time models |
| Ledger | Postgres (immutable, double-entry) | See [Payment System](payment-system.md) |

### Consistency Choices

- **Order + payment**: strong (single-shard transaction + outbox). Never an eventually-consistent store for order status.
- **Search/menu/availability**: eventual; reconcile at order time.
- **DP location**: best-effort; staleness bounded by heartbeat TTL (> 15 s stale ⇒ not assignable).
- **Assignment**: CAS on both DP and order so there is exactly one DP per order and one order slot per DP.

### Multi-Region & Disaster Recovery

- Region per geography (India: Mumbai primary + Hyderabad). Cities pinned to a region — no cross-region calls on the hot path.
- Active-active across regions with city partitioning; async cross-region replication of order data for DR (RPO seconds); cutover runbook for a region failure.
- Global services (user accounts, restaurant onboarding) replicated read-mostly.
- DNS + health-checked routing; clients retry the second region on failure; order idempotency keys make retries safe.

### Observability

- Golden signals per service plus **business SLIs**: order success rate, time-to-accept, time-to-assign, ETA error, DP idle %, cancellation rate by reason, cold-food rate.
- Distributed tracing with `order_id` as a baggage tag; one click to see an order's full saga.
- Stuck-order detector: orders in a non-terminal state beyond its SLA (e.g., `PLACED > 3 min`) fire alerts and auto-remediation.
- Real-time dashboards by city/zone (ClickHouse/Druid), anomaly alerts on orders/min drop (a payment-provider outage shows up as orders/min cliff).

### Failure Modes

| Failure | Handling |
|---------|---------|
| Payment provider down | Fail over to a second PSP; show alternative methods; COD fallback; circuit breaker |
| Restaurant tablet offline | Heartbeat lost → close restaurant in search; escalate IVR call; reject/timeout → refund |
| Order Service crash mid-saga | Saga state in DB; recovery worker resumes orders stuck in intermediate states |
| Duplicate order submission (retry) | Idempotency key returns original order |
| Kafka consumer lag | Autoscale consumers; lag alerts; DP offers expire safely, orders still advance via CAS |
| DP app loses network | Last location retained; after TTL not assignable; in-trip → ops contact + re-assign if not picked up |
| DP rejects / ignores offers | Re-run assignment next cycle; escalate incentive; widen radius |
| Redis geo index lost | Rebuild from the next ping wave (≤ 15 s); fall back to Postgres lookups for in-flight orders |
| Hot restaurant (viral deal) | Per-restaurant order cap, queue/back-pressure, "busy" state with longer ETA |
| Region failure | Cell failover to peer region; replay outbox; idempotency prevents double effects |
| ETA model service down | Fallback to rule-based ETA (static prep + routing) |

### Evolution (Interview Roadmap)

1. **MVP**: monolith, Postgres + PostGIS, nearest-DP greedy, polling for tracking.
2. **Scale**: split Order/Dispatch/Search, Kafka, Redis geo, Elasticsearch, WebSockets.
3. **Optimise**: batched assignment, prep-time ML, surge, order batching.
4. **Platform**: multi-vertical (grocery, dine-out), ads marketplace, cell-based architecture, own fleet management.

### Common Mistakes

| Mistake | Fix |
|---------|-----|
| Assigning the nearest DP at accept time | Predict food-ready time; batch + dispatch just-in-time |
| Treating distance as serviceability | Road-distance + capacity-aware serviceability cells |
| Storing live menu availability in the catalogue DB | Separate Redis overlay with its own change stream |
| Synchronous saga across payment, restaurant, dispatch in one request | Orchestrated async saga with timeouts and compensations |
| No timeout on restaurant acceptance | Delay queue + idempotent timeout handler + refund |
| Pushing every DP ping to every service | Kafka fan-out; route only to the customer of the active order |
| Ignoring kitchen capacity | Per-restaurant concurrency caps and dynamic ETAs |
| Mutating orders without version/CAS | State-machine transitions with compare-and-set + outbox |
| Mixing browse and order SLOs | CQRS: cached AP read path, strongly consistent order path |
| Refund as UPDATE on the original payment row | New compensating ledger entries |

> [!NOTE]
> The three hard problems: (1) **assignment** — a continuous batched optimisation tied to prep-time prediction; (2) **the saga** — three independent parties each of which can say no or vanish; (3) **spikes** — the kitchen, not the server, is the real bottleneck.

### Low-Level Design Appendix (OOD Round)

Based on the common LLD problem "Food Delivery Service" (see [ashishps1/awesome-low-level-design](https://github.com/ashishps1/awesome-low-level-design/blob/main/problems/food-delivery-service.md)).

**Requirements**: customers browse restaurants/menus, add to cart, place order; restaurants manage menus; delivery agents get assigned; order status tracked and customers notified; cancellation supported.

**Classes**:
- `Customer(id, name, address)`
- `Restaurant(id, name, address, List<MenuItem> menu)`
- `MenuItem(id, name, price, isAvailable)`
- `Order(id, customer, restaurant, List<OrderItem>, status, deliveryAgent)`
- `OrderItem(menuItem, quantity)`
- `DeliveryAgent(id, name, location, isAvailable)`
- `FoodDeliveryService` (singleton facade): `placeOrder`, `cancelOrder`, `updateOrderStatus`, `assignAgent`
- `enum OrderStatus { PENDING, CONFIRMED, PREPARING, OUT_FOR_DELIVERY, DELIVERED, CANCELLED }`

**Patterns**:
- **State** — each `OrderStatus` has behaviour (what transitions are legal; cancel allowed only before OUT_FOR_DELIVERY).
- **Strategy** — `AgentAssignmentStrategy` (nearest, least-loaded, rating-weighted) and `PricingStrategy` (flat, surge) swappable at runtime.
- **Observer** — `OrderObserver` (customer app, restaurant, agent) notified on every status change.
- **Singleton / Facade** — `FoodDeliveryService` is the entry point.

```java
public enum OrderStatus { PENDING, CONFIRMED, PREPARING, OUT_FOR_DELIVERY, DELIVERED, CANCELLED }

public interface OrderObserver { void onStatusChange(Order order, OrderStatus newStatus); }

public interface AgentAssignmentStrategy {
    Optional<DeliveryAgent> pick(Order order, Collection<DeliveryAgent> agents);
}

public class NearestAgentStrategy implements AgentAssignmentStrategy {
    public Optional<DeliveryAgent> pick(Order o, Collection<DeliveryAgent> agents) {
        return agents.stream()
            .filter(DeliveryAgent::isAvailable)
            .min(Comparator.comparingDouble(a -> a.getLocation().distanceTo(o.getRestaurant().getLocation())));
    }
}

public class Order {
    private final String id;
    private final Customer customer;
    private final Restaurant restaurant;
    private final List<OrderItem> items;
    private OrderStatus status = OrderStatus.PENDING;
    private DeliveryAgent agent;
    private final List<OrderObserver> observers = new CopyOnWriteArrayList<>();

    private static final Map<OrderStatus, Set<OrderStatus>> ALLOWED = Map.of(
        OrderStatus.PENDING,          Set.of(OrderStatus.CONFIRMED, OrderStatus.CANCELLED),
        OrderStatus.CONFIRMED,        Set.of(OrderStatus.PREPARING, OrderStatus.CANCELLED),
        OrderStatus.PREPARING,        Set.of(OrderStatus.OUT_FOR_DELIVERY, OrderStatus.CANCELLED),
        OrderStatus.OUT_FOR_DELIVERY, Set.of(OrderStatus.DELIVERED),
        OrderStatus.DELIVERED,        Set.of(),
        OrderStatus.CANCELLED,        Set.of());

    public synchronized void transition(OrderStatus next) {
        if (!ALLOWED.get(status).contains(next))
            throw new IllegalStateException(status + " -> " + next + " not allowed");
        this.status = next;
        observers.forEach(o -> o.onStatusChange(this, next));
    }

    public double total() {
        return items.stream().mapToDouble(i -> i.getMenuItem().getPrice() * i.getQuantity()).sum();
    }
}

public class FoodDeliveryService {
    private static final FoodDeliveryService INSTANCE = new FoodDeliveryService();
    private final Map<String, Order> orders = new ConcurrentHashMap<>();
    private final Collection<DeliveryAgent> agents = new CopyOnWriteArrayList<>();
    private AgentAssignmentStrategy strategy = new NearestAgentStrategy();
    public static FoodDeliveryService getInstance() { return INSTANCE; }

    public Order placeOrder(Customer c, Restaurant r, List<OrderItem> items) {
        items.forEach(i -> { if (!i.getMenuItem().isAvailable()) throw new IllegalArgumentException("Unavailable"); });
        Order order = new Order(UUID.randomUUID().toString(), c, r, items);
        order.addObserver(new CustomerNotifier(c));
        orders.put(order.getId(), order);
        return order;
    }

    public void confirm(String orderId) {
        Order o = orders.get(orderId);
        o.transition(OrderStatus.CONFIRMED);
        strategy.pick(o, agents).ifPresent(a -> { a.setAvailable(false); o.assign(a); });
    }

    public void cancel(String orderId) {
        Order o = orders.get(orderId);
        o.transition(OrderStatus.CANCELLED);
        if (o.getAgent() != null) o.getAgent().setAvailable(true);
    }
}
```

LLD follow-ups: *how to make assignment thread-safe* (CAS/lock on the agent's availability flag, `AtomicBoolean`), *how to add a new status* (extend enum + transition table; or move to State classes), *how to add new pricing* (new Strategy, no change in service), *how to support scheduled orders* (add `scheduledAt` + a scheduler that moves to CONFIRMED at the right time).

### Interview Follow-ups

- *"How do you find restaurants that deliver to a user?"* — Invert the relation: index each restaurant with the H3 cells it serves; query by user's cell as a term filter in Elasticsearch, combined with text relevance and ranking. Serviceability is recomputed from road distance, DP supply and kitchen load.
- *"Why not just geo_distance within 5 km?"* — Straight-line radius ignores rivers, one-way roads, congested zones, and capacity; and a precomputed cell filter is cheaper at 15k QPS.
- *"Menu changes constantly — how do you keep search results accurate?"* — Split catalogue (slow) from availability (fast). Availability is an overlay in Redis updated by events; ES updated near-real-time for "is_open" and dish stock; order-time check is authoritative.
- *"How do you prevent ordering an item that's out of stock?"* — Order-time validation against the authoritative availability store; restaurant can reject; auto-disable items with high rejection rates; show "sold out" quickly via overlay.
- *"How do you make order placement idempotent?"* — Client-generated `Idempotency-Key`, unique constraint `(customer_id, key)`; replay returns the original order; same for the payment call to the PSP.
- *"What if payment succeeds but the restaurant never responds?"* — A scheduled timeout moves the order to TIMED_OUT via CAS, voids/refunds the payment, and notifies the customer with alternatives.
- *"What if the restaurant accepts at the same moment the timeout fires?"* — Both do CAS on status; exactly one wins; the loser no-ops. The timeout handler re-reads state.
- *"Saga orchestration or choreography?"* — Orchestration (Order Service) for this flow: many steps, timeouts, compensations, observability. Services still communicate via events; choreography is fine for loosely-coupled side effects (notifications, analytics).
- *"How do you guarantee an event is published when the order row changes?"* — Transactional outbox: write the event to an `outbox` table in the same transaction, a relay (Debezium/poller) publishes to Kafka. Consumers dedupe by event id.
- *"How do you assign a delivery partner?"* — Batched min-cost assignment per zone every few seconds, with cost including DP travel, food-wait, DP-idle and customer delay; offers with TTL; CAS on both DP and order. Greedy nearest is the MVP.
- *"Why not dispatch as soon as the restaurant accepts?"* — The DP would wait idle at the restaurant. Dispatch at `predicted_ready − DP_travel_time`; if supply is thin, dispatch early.
- *"How do you predict preparation time?"* — Quantile GBDT on restaurant, item mix, live backlog, daypart, weather; ground-truth from DP pickup time rather than the restaurant's READY tap; p80 for promises.
- *"How do you batch multiple orders to one DP?"* — Candidate bundles with the same restaurant or nearby ones and compatible directions; accept the bundle if detour adds less than a threshold per customer and total cost beats separate assignment.
- *"Two DPs accept the same offer at once?"* — Atomic CAS `order.dp_id IS NULL → dp_id`; loser receives "taken" and returns to the pool. Offers are also sent to a limited number of DPs to reduce collisions.
- *"How does real-time tracking scale?"* — DP pings → Kafka → latest in Redis; Tracking Service maps `dp → order` and pushes via a WebSocket/SSE gateway only to the interested customer; client-side interpolation allows lower frequency; adaptive ping rate.
- *"WebSocket connections drop on mobile — what then?"* — Client reconnects with last-event-id, gets the latest state from Redis; fallback to short polling or FCM; the UI stays correct because state is server-owned.
- *"How is the ETA computed and kept honest?"* — Sum of accept, prep p80, DP-to-restaurant, handover and road-routing legs; recomputed live; compare to actuals, publish ETA-error metrics per cell; adjust with a buffer and never silently stretch.
- *"How do you compute surge/delivery fee?"* — Demand/supply ratio per cell over a sliding window with smoothing and caps; locked into a signed price token at checkout; prefer DP incentives and fee levers over surprise charges.
- *"Customer's price changed between cart and payment — what do you do?"* — Price token with expiry; if expired or mismatched, re-quote and ask for re-confirmation. Orders store a price snapshot so later changes never affect them.
- *"How would you rank search results?"* — Learning-to-rank blend of relevance, conversion, Bayesian rating, predicted ETA, availability, personalisation, minus complaint rate; guard against closed/overloaded restaurants; exploration for new restaurants; ads labelled.
- *"How do you handle IPL-final traffic?"* — Pre-scale from the match schedule; cache and shed non-critical load; per-restaurant order caps; incentive boosts for DPs; cell-based isolation; prioritised queues; waiting-room admission control.
- *"Won't autoscaling be too slow for a 2-minute spike?"* — Yes. Use scheduled/predictive scaling plus warm headroom, pre-warmed caches and connection pools, and load shedding while capacity ramps.
- *"How do you prevent a restaurant from being overloaded?"* — Track open orders vs configured capacity; auto-pause or extend prep estimate; reduce serviceability radius; surface "busy" in search.
- *"How do you handle refunds without double-refunding?"* — Idempotent per `(order_id, reason)`; refunds are ledger entries; PSP refund call uses an idempotency key; reconciliation catches mismatches.
- *"How would you stop a customer who claims 'not delivered' repeatedly?"* — Risk score on refund rate, DP GPS + OTP evidence, device/account graph, manual review thresholds, progressive friction (instead of automatic refund).
- *"How do you stop a DP faking GPS or deliveries?"* — Mock-location detection, geofence at drop, OTP handover, speed/jump anomalies, photo proof for no-contact delivery, cross-check with customer phone proximity.
- *"How do you keep coupon budgets correct under concurrency?"* — Atomic Redis counters (`DECR` with floor), reservation at checkout released on failure/expiry, reconciled to DB; per-user caps via unique constraints.
- *"How do you shard orders?"* — By `customer_id` for history queries (user-centric) with a global secondary mapping by `order_id`, or by `order_id` with separate read models for restaurants and DPs. Hot restaurants are handled in the read model, not the write shard.
- *"How do restaurants see their orders in real time and reliably?"* — Dedicated restaurant read model; push + WebSocket + alarm; IVR fallback; acknowledgement tracking with retries; auto-accept via POS integration.
- *"How do you do multi-region?"* — Cities pinned to a region; active-active across cities; async replication for DR; idempotency keys make retried requests safe on failover; global data (users) replicated.
- *"What happens during a Kafka outage?"* — Orders still commit to the DB with the outbox row; relay catches up on recovery; a synchronous fallback path for critical notifications (direct push) keeps customers informed.
- *"How do you test dispatch changes safely?"* — Offline replay simulation with historical orders/DP traces, then shadow mode, then city-level A/B with guardrail metrics (delivery time, cold food, DP idle, cost per order).
- *"How do you observe stuck orders?"* — Per-state SLA timers and a sweeper job; alerts when counts in non-terminal states exceed thresholds; traces keyed by `order_id`; auto-remediate (re-dispatch, cancel + refund).
- *"How does the cost per order look and where do you save?"* — Biggest costs: delivery payouts, discounts, payment fees, support. Levers: order batching, better ETA (less DP idle), restaurant-prep prediction, fewer refunds through quality signals, fee tuning, cheaper PSP routing.
- *"How would you add grocery / quick commerce?"* — Different model: dark-store inventory with strict stock counts (see [Inventory Management](inventory-management.md)), pick-pack workflows, 10-minute promises, smaller radius, pre-positioned DPs.
- *"How would you support scheduled orders?"* — Store `scheduled_for`; a scheduler creates the order saga at `scheduled_for − prep − travel`; validate availability again at trigger time; allow free cancellation until then.
- *"How do you handle GDPR/DPDP data requests and location privacy?"* — Location retained only as long as needed, anonymised for analytics, phone numbers masked via proxy calling, delete/export workflows keyed by user id.
- *"What would you monitor on day one?"* — Order success rate, payment success, time-to-accept, time-to-assign, ETA error, cancellation by reason, DP utilisation, tracking lag, queue depths, search latency.
