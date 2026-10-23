# Q: Design an online shopping platform (Amazon).

**Answer:**

An e-commerce platform is a **collection of independent, domain-owned services** (catalog, search, cart, pricing, inventory, orders, payments, fulfillment, reviews, recommendations) glued together by events. The interview tests **where to be strongly consistent** (inventory, orders, payments) versus **where eventual consistency is a feature** (catalog, search, reviews, recommendations), plus **flash-sale/Prime-Day scaling** and **oversell prevention**.

Amazon's own story is instructive: the 2007 Dynamo paper came from the *shopping cart* requirement of "always writable, even during partitions" — merge conflicts rather than reject writes. That drives the cart design below.

### Requirements

**Functional**:
- Browse and search the catalog (millions of products, variants, sellers).
- Product detail page (PDP): images, price, availability, delivery promise, reviews, recommendations.
- Cart (guest and logged-in; merge on login).
- Checkout: address, shipping option, payment, promo codes.
- Place order, order history, order tracking, cancellations, returns.
- Reviews and ratings; Q&A.
- Personalised recommendations ("customers also bought").
- Third-party sellers (marketplace) with their own inventory and fulfilment.
- Sales events: lightning deals, Prime Day, Black Friday.

**Non-functional**:
- Catalog / PDP / search: very high availability (99.99%), p99 < 200 ms, eventually consistent.
- Cart: always writable (availability over consistency).
- Checkout + orders + inventory + payment: correctness over availability; **never oversell, never double-charge**.
- 10x traffic spikes during events with a long-tail of hot SKUs.
- Global, multi-region; localisation (currency, tax, language).

**Out of scope (state it)**: seller onboarding/billing details, ads auction, returns logistics details, detailed fraud ML.

### Capacity Estimation

- 300M MAU, 100M DAU. Peak shopping event: 10x normal.
- Page views: 100M DAU × 30 views = 3B/day = **35k QPS avg**, **~350k QPS peak** (mostly cacheable).
- Search: ~20% of sessions → **~10k QPS avg**, 100k+ peak.
- Orders: 20M/day avg = ~230/sec; Prime Day peak ~**5–10k orders/sec** globally.
- Catalog: 500M products (incl. variants/offers) × 10 KB = **5 TB** metadata; images 100x larger (CDN/S3, tens of PB).
- Cart writes: add-to-cart ≈ 5x orders → ~**50k writes/sec** at peak.
- Inventory: 500M SKU-location rows; hot subset of SKUs under heavy concurrent decrement.
- Reviews: 2B reviews × 1 KB = 2 TB; read-heavy.

Observation: reads outnumber writes ~1000:1 for catalog; writes concentrate on cart and on a handful of hot inventory rows.

### API Design

```
GET  /v1/search?q=wireless+earbuds&filters=brand:sony,price:<5000&sort=relevance&cursor=
GET  /v1/products/{asin}                         # PDP aggregate (BFF composes)
GET  /v1/products/{asin}/offers                  # sellers / buy-box
POST /v1/cart/items          { sku, qty }        # works for guest via cart cookie
PUT  /v1/cart/items/{id}     { qty }
GET  /v1/cart
POST /v1/checkout/session    { cart_id, address_id, shipping_option }   → priced quote + reservation
POST /v1/orders              Idempotency-Key: <uuid>   { checkout_session_id, payment_method }
GET  /v1/orders?cursor=      # history
GET  /v1/orders/{id}/tracking
POST /v1/orders/{id}/cancel | /return
POST /v1/products/{asin}/reviews   { rating, title, body, order_id }
GET  /v1/recommendations?context=pdp|cart|home
```

### High-Level Architecture

```
          Browser/App ──► CDN (images, static, cached PDP fragments)
                              │
                         API Gateway / BFF (page composition)
                              │
   ┌────────────┬─────────────┼──────────────┬──────────────┬───────────────┐
   ▼            ▼             ▼              ▼              ▼               ▼
 Search      Catalog        Cart          Pricing &      Reviews        Recommend-
 (ES/Solr)   Service       Service        Promotions     Service         ations
   ▲            │             │              │              │               │
   │ index      ▼             ▼              ▼              ▼               ▼
   └──── Kafka ◄── events ───────────────────────────────────────────────────┘
                              │
             Checkout / Order Service (saga orchestrator)
              │          │            │             │
              ▼          ▼            ▼             ▼
         Inventory   Payment      Tax/Shipping   Fulfillment (WMS) ─► Carrier ─► Tracking
         Service     Service      Calculators    (warehouse routing)
              │          │
              ▼          ▼
        Inventory DB   Ledger / PSP
        (strongly consistent)
```

**Service-per-domain**: each service owns its data store; no shared DB. Cross-domain data flows through events (Kafka/SNS+SQS) and API calls. This is the "two-pizza team, you build it you run it" model from Amazon's 2002 mandate.

**CQRS**: write models (catalog admin, inventory, orders) are normalised and consistent; read models (search index, PDP cache, order-history view, recommendations) are denormalised projections built from events and can lag by seconds.

### Catalog Service

- **Model**: product (parent) → variants (size/colour SKUs) → offers (seller, price, condition). The "buy box" picks the winning offer.
- Attributes are heterogeneous (shoes vs laptops): store as a flexible document (DynamoDB/MongoDB/Postgres JSONB) with category-specific schemas validated at ingestion.
- Source of truth: catalog DB. Everything else (search, PDP cache) is derived.
- Ingestion: sellers upload by API/feeds → validation pipeline (schema, image checks, moderation, dedupe/matching to existing ASIN) → catalog write → change event.
- Images: S3 + CDN; multiple renditions generated asynchronously.

```sql
-- illustrative relational shape
CREATE TABLE product (asin TEXT PRIMARY KEY, title TEXT, brand TEXT, category_id BIGINT,
                      attributes JSONB, status TEXT, updated_at TIMESTAMPTZ);
CREATE TABLE sku     (sku TEXT PRIMARY KEY, asin TEXT REFERENCES product, variant JSONB);
CREATE TABLE offer   (offer_id BIGINT PRIMARY KEY, sku TEXT, seller_id BIGINT,
                      price NUMERIC(12,2), currency CHAR(3), condition TEXT, fulfilled_by TEXT);
```

At Amazon scale this is DynamoDB (key-value by ASIN) rather than a monolithic relational DB — partition key `asin`, read via aggressive caching.

### Search (Elasticsearch / OpenSearch)

```
Catalog change ─► Kafka ─► Indexer ─► Elasticsearch (sharded, replicated)
Query ─► Query understanding (spell, synonyms, intent, category prediction)
      ─► ES retrieve (BM25 + filters + facets) ─► ML re-ranker (LTR) ─► blend ads ─► results
```

- Index document: title, brand, category path, attributes, price, rating, review count, sales rank, availability, delivery speed, seller type.
- **Facets** (brand, price range, rating) via ES aggregations; cached for popular queries.
- Ranking: text relevance × (conversion, sales velocity, review quality, price competitiveness, Prime eligibility, in-stock, delivery promise, personalisation). Learning-to-rank with click/purchase logs.
- **Availability and price in search**: indexed with eventual consistency (seconds); PDP and cart re-validate against authoritative services. Never trust search for the final price.
- Sharding: by category/product hash; replicas for read QPS; time-based or alias-based zero-downtime reindex.
- Pagination: `search_after` cursors, not deep `from/size`.
- Autocomplete: separate index/trie. See [Typeahead](typeahead.md).
- Scale knobs: query cache, request coalescing, per-locale indexes, degrade by dropping the re-ranker under load.

### Product Detail Page (PDP)

PDP is an **aggregation of ~20 services** (catalog, price, availability, delivery date, reviews summary, recommendations, ads, Q&A).

- BFF fans out in parallel with **tight per-call timeouts** and fallbacks; each widget is optional except title/price/buy.
- **Cache layers**: CDN caches the static portion (title, images, description), Redis/DAX for hot product docs, per-widget cache with short TTLs.
- Page streams progressively: critical above-the-fold first, long-tail widgets lazy-loaded.
- Personalised parts (price with coupons, delivery to my address, recs) are loaded client-side so the HTML itself is cacheable.

> [!NOTE]
> The PDP principle: **degrade gracefully**. If recommendations or reviews are slow, drop them; never fail the page because a non-critical dependency is down.

### Shopping Cart

Requirements: always writable, survive partitions, support guest carts, merge on login, scale to ~50k writes/sec.

**Options**:

| Option | Pros | Cons |
|--------|------|------|
| Redis (hash per cart, TTL) | Sub-ms, simple, atomic `HINCRBY` | Durability concerns; needs replication + persistence; memory cost |
| DynamoDB (cart_id → item map) | Durable, multi-AZ/global tables, infinite scale, always-writable | Item size limit 400 KB; conflict semantics for multi-region |
| Relational | Familiar | Hot rows, vertical scaling, poor fit for TTL/guest volume |

**Production answer**: **DynamoDB (or Cassandra) as the durable store, Redis in front as a cache** for hot active carts. Cart item = `(sku, qty, added_at, price_seen)`.

```
PK cart_id   (guest: cart cookie id; user: user_id)
attributes: items{ sku → {qty, added_at, seller_id, price_seen} }, updated_at, ttl
```

**Guest vs logged-in**:
- Guest: cart_id cookie (random UUID, 30–90 day TTL).
- On login: **merge** guest cart into user cart. Merge rule: union of SKUs; for duplicates take `max(qty)` or sum (product choice); deleted-in-one-side issues resolved with per-item tombstones + timestamps.
- Dynamo's historical choice: on conflicting concurrent writes (from partitions) keep **both versions and merge by union**, which means a removed item can occasionally reappear — acceptable for carts, unacceptable for money.

**Cart does not reserve inventory.** Reserving at add-to-cart would let abandoned carts lock stock. The cart shows "only 2 left" as a hint; the reservation happens at checkout.

**Price/stock changes in the cart**: show "price changed since you added" and re-validate at checkout from authoritative services.

### Pricing & Promotions

- Pricing service computes: base price (seller offer), list price strike-through, dynamic repricing (seller algorithms/competition), coupons, Prime/lightning deal prices, taxes, shipping fees.
- **Deals** are time-boxed overrides with their own inventory cap (e.g., 500 units at deal price).
- Pricing computed at: PDP (display), cart (estimate), checkout (binding quote with expiry). The binding quote is stored with the order as a snapshot.
- Price history and "was price" regulatory compliance (store immutable price-change events).
- Cache aggressively; invalidate by events; accept seconds of staleness on display, never at checkout.

### Checkout Flow

```
1. POST /checkout/session
     - re-price every line (authoritative), compute tax + shipping per warehouse split
     - RESERVE inventory for each line (TTL ~10–15 min)   ← strongly consistent
     - return quote + reservation_id + expiry
2. User confirms → POST /orders (Idempotency-Key)
     - verify reservation still valid
     - payment authorise (PSP) → on success create order (PLACED)
     - convert reservation → committed allocation
3. Emit order_placed → fulfilment, notifications, recommendations, analytics
```

Reservation at **checkout start** (not add-to-cart, not payment complete) is the standard trade-off: long enough to prevent "paid but out of stock", short enough to not strangle inventory with abandoners.

### Inventory Reservation & Oversell Prevention

The core consistency problem. Multiple buyers contend for the last unit.

Inventory record per `(sku, warehouse)`:

```sql
CREATE TABLE inventory (
    sku          TEXT,
    warehouse_id INT,
    on_hand      INT NOT NULL,
    reserved     INT NOT NULL DEFAULT 0,
    version      BIGINT NOT NULL DEFAULT 0,
    PRIMARY KEY (sku, warehouse_id),
    CHECK (on_hand - reserved >= 0)
);

-- atomic conditional reserve (no read-modify-write race)
UPDATE inventory
   SET reserved = reserved + :qty, version = version + 1
 WHERE sku = :sku AND warehouse_id = :wh AND on_hand - reserved >= :qty;
-- rows_affected = 0  →  insufficient stock → reject
```

Equivalent in DynamoDB: `UpdateItem` with `ConditionExpression: on_hand - reserved >= :qty` — single-item atomic operation, the idiom Amazon uses. In Redis: Lua script doing check-and-decrement.

Reservation record:

```sql
CREATE TABLE reservation (
    id UUID PRIMARY KEY, order_ref TEXT, sku TEXT, warehouse_id INT, qty INT,
    status TEXT,                 -- HELD, COMMITTED, RELEASED, EXPIRED
    expires_at TIMESTAMPTZ
);
```

- **Expiry**: a sweeper (or Redis keyspace/TTL queue / delay queue) releases `HELD` reservations past `expires_at` and increments availability back.
- **Commit** on successful payment: `reserved -= qty; on_hand -= qty` (actual allocation), idempotent per reservation id.
- **Release** on cancel/failure; idempotent.
- **Soft vs hard allocation**: soft = counts toward `reserved`; hard = physical stock assigned to an order at the warehouse when picked.

**Hot SKU problem (flash sale)**: thousands of concurrent decrements on one row.
- Row-level lock / conditional update on one DynamoDB item caps at ~1,000 WCU/partition. Solutions:
  1. **Sharded counters**: split stock 1000 → 10 buckets of 100; pick a random bucket; fall back to others when empty. Total available = sum (eventually consistent for display).
  2. **Redis pre-decrement gate** (atomic `DECR`/Lua) as admission control in front of the DB: only winners proceed to the durable reservation; losers get "sold out" immediately. Reconcile Redis counter with DB.
  3. **Queue-based serialisation** for ultra-hot SKUs: partition by SKU into a single-writer queue; the consumer allocates in order (fair FIFO, trivially no oversell).
- See [Inventory Management](inventory-management.md) for the deep dive.

**Oversell trade-off**: some businesses allow controlled oversell (backorder) with safety stock buffers when stock is replenished regularly and cancellation is cheap; for limited drops, strictly never.

### Order Service & Saga

```
Order states:  CREATED → PAYMENT_AUTHORISED → CONFIRMED → ALLOCATED → PACKED → SHIPPED → DELIVERED
                 ↘ CANCELLED / PAYMENT_FAILED / OUT_OF_STOCK        ↘ RETURN_REQUESTED → REFUNDED
```

Orchestrated saga:

| Step | Service | Compensation |
|------|---------|--------------|
| 1 Reserve stock | Inventory | Release reservation |
| 2 Authorise payment | Payment | Void authorisation |
| 3 Create order (CONFIRMED) | Order | Mark CANCELLED |
| 4 Allocate warehouse / create fulfilment | Fulfilment | Cancel fulfilment request |
| 5 Capture payment (at ship) | Payment | Refund |

- State persisted in the order DB; a **recovery worker** resumes stuck sagas.
- Transactional **outbox** for reliable event publishing.
- Orders split into **shipments** (items from different warehouses/sellers) each with its own state; order status is derived from the shipment states.
- **Capture at shipment** (not at order time) is Amazon's classic behaviour — authorise at order, charge when it ships; avoids refunds for cancelled-before-ship.

See [Payment System](payment-system.md) for ledger/idempotency.

```sql
CREATE TABLE orders (
    order_id BIGINT PRIMARY KEY, user_id BIGINT, status TEXT, total NUMERIC(12,2),
    currency CHAR(3), price_snapshot JSONB, address_snapshot JSONB,
    version INT, created_at TIMESTAMPTZ
);
CREATE TABLE order_lines (order_id BIGINT, line_no INT, sku TEXT, qty INT, unit_price NUMERIC(12,2),
                          seller_id BIGINT, shipment_id BIGINT, PRIMARY KEY(order_id,line_no));
CREATE INDEX ON orders (user_id, created_at DESC);
```

### Payments

- Dedicated Payment service (PCI scope isolated); tokenised cards; PSP abstraction layer to route across providers.
- **Authorise → capture → refund**, each idempotent with keys.
- Multiple payment methods (cards, wallets, EMI/BNPL, gift balance) → split payments need an atomic multi-instrument flow with reversals.
- 3-D Secure/SCA step handled via redirect and webhook; order stays `PAYMENT_PENDING` until confirmation, with a timeout.
- Ledger entries for every money movement; daily reconciliation with PSP.

### Order History & Tracking (CQRS read model)

- Order history is read far more than written: build a **denormalised order-view** (DynamoDB: PK `user_id`, SK `created_at#order_id`) updated by order/shipment events.
- Avoids joins across Order, Shipment, Product services; product name/image snapshots stored on the order.
- Shipment tracking: carrier webhooks/polling → Shipping service → events → order view + push notifications. Normalise statuses across carriers (label created, picked up, in transit, out for delivery, delivered, exception).
- Delivery promise (date) computed at checkout from warehouse location, cut-off times, carrier SLAs — stored on the order; updated on exceptions.

### Fulfilment & Warehouses

- **Warehouse selection**: given destination + items, choose the fulfilment node(s) that minimise cost and meet promise date; weights: stock availability, distance/zone, capacity, split-shipment penalty.
- WMS (warehouse management) handles inbound, putaway, pick, pack, ship. Inventory events from WMS feed the Inventory service (receipts, adjustments, damages).
- Marketplace modes: FBA (Amazon-fulfilled — Amazon holds the stock), FBM (seller-fulfilled — seller's own inventory feed, with stale-stock risk and cancellations).
- Demand forecasting positions stock near customers (pre-positioning) for fast delivery promises.

### Reviews & Ratings

- Write path: only verified purchasers flagged "Verified Purchase" (check `order_id`); one review per purchase.
- Pipeline: submit → spam/fake/abuse detection (ML + rules) → moderation → publish → async aggregation.
- Aggregates (avg rating, histogram, count) maintained by stream processors in a separate store; PDP reads the precomputed summary, not raw reviews.
- Ranking of reviews ("most helpful") by helpful votes, recency, verified, with anti-gaming.
- Reviews are **eventually consistent** and can be stored in DynamoDB/Cassandra (product_id partition, sorted by helpfulness/time) with ES for text search.

### Recommendations

- Offline (batch): item-to-item collaborative filtering (Amazon's signature approach: "customers who bought X also bought Y"), matrix factorisation, embeddings.
- Nearline: session-based/real-time signals from clicks (Kafka → feature store).
- Online serving: candidate retrieval (ANN over embeddings) → ranking model → business rules (availability, margin, diversity).
- Precompute top-N per product/user into a KV store for sub-10 ms serving; fall back to popular items for cold-start.
- See [Recommendation System](recommendation-system.md).

### Flash Sales / Prime Day

Challenges: sudden 10–50x spikes on a few SKUs, bots, inventory contention, payment bursts.

1. **Pre-warm & pre-scale**: capacity planning, game days/load tests, scale caches and DB partitions beforehand (Amazon's "scale up before the event").
2. **Static the hot page**: deal PDP served from CDN, countdown driven client-side, "Add to cart" hits only the cart/reservation path.
3. **Admission control / virtual waiting room**: queue users at the edge (token bucket, signed tokens, randomised draw) so the backend sees a bounded rate.
4. **Redis inventory gate** for deal units (atomic Lua decrement), async durable reservation, immediate sold-out feedback.
5. **Bot protection**: CAPTCHAs, device fingerprinting, per-account purchase limits (max qty per customer), rate limits per IP/account.
6. **Load shedding by priority**: protect cart → checkout → payment; drop recommendations/ads/analytics first.
7. **Queue write-heavy flows** (order placement behind a queue with a "we're processing" state) when payment/DB saturates.
8. **Cell-based architecture** so a failure affects a subset of customers.
9. **Async everything non-critical**: emails, invoices, recommendation updates.
10. Idempotent order endpoints so retry storms are harmless.

### Eventual Consistency — Where and Why

| Domain | Consistency | Notes |
|--------|-------------|-------|
| Catalog, search index | Eventual (seconds–minutes) | Derived from events |
| Reviews, recommendations | Eventual | Aggregates lag |
| Cart | Eventual / always-writable | Merge conflicts by union |
| Price on PDP | Eventual | Re-validated at checkout |
| Inventory reserve | **Strong (single-item atomic)** | Conditional update |
| Order, payment | **Strong** | ACID per aggregate |
| Order history view | Eventual | Read-your-writes handled by session pinning or write-through |

**Read-your-writes** for "I just placed an order but it's not in my history": write the new order into the view synchronously (or return it from the POST and have the client merge), or query the primary for a short window after writes.

### DynamoDB Origins (what to know)

- Dynamo (2007) built for the shopping cart: highly available, partitioned by consistent hashing, replicated N ways, **sloppy quorum**, **vector clocks**, **hinted handoff**, **read repair**, **anti-entropy via Merkle trees**.
- DynamoDB (the managed service) is a different system with the same lineage: partition key → partitions, leader-based replication, optional strongly consistent reads, conditional writes, transactions (`TransactWriteItems`), streams (CDC feeding indexers and read models), global tables (multi-region, last-writer-wins).
- Use patterns: single-table design with PK/SK, GSIs for alternate access paths, TTL for expiring carts/reservations.
- See [Distributed Cache](distributed-cache-design.md) for consistent hashing background.

### Storage Choices

| Data | Store | Why |
|------|-------|-----|
| Product catalog | DynamoDB / Aurora + S3 for media | Key access by ASIN, huge scale |
| Search index | Elasticsearch / OpenSearch | Full-text, facets, LTR |
| Cart | DynamoDB + Redis | Always-writable, TTL, hot cache |
| Inventory | DynamoDB / sharded Postgres | Atomic conditional updates |
| Orders, payments | Aurora/Postgres (sharded) or DynamoDB with transactions | Strong consistency per aggregate |
| Order history view | DynamoDB | Per-user partition, time-ordered |
| Reviews | DynamoDB / Cassandra + ES | Write heavy, partition by product |
| Recommendations | Feature store + KV (Redis/DynamoDB) | Precomputed lists |
| Events | Kafka / Kinesis / SNS-SQS | Decoupling, CQRS projections |
| Images/video | S3 + CloudFront | Cheap blob + CDN |
| Analytics | S3 data lake + Redshift/Athena/ClickHouse | OLAP, ML training |
| Ledger | Immutable double-entry store | Money |

### Multi-Region

- Active-active regions; users routed to the nearest; **orders homed** in one region per customer (avoids cross-region write conflicts), with async replication for DR.
- Catalog/search replicated read-only to all regions (events fan out).
- Inventory is region/warehouse-local (stock physically lives somewhere); a global view is a read model.
- Currency/tax/legal localisation per marketplace; data-residency constraints dictate where PII lives.
- Failover: DNS/Route53 health checks; cell isolation; idempotent writes make client retries on the second region safe.

### Observability

- Business SLIs: add-to-cart rate, checkout conversion, payment success, order success, oversell incidents (should be 0), search zero-result rate, p99 PDP.
- Trace IDs across the PDP fan-out and the checkout saga; dashboards per dependency with SLO burn alerts.
- Canary/gradual deploys with automatic rollback on conversion regressions.
- Synthetic transactions (a "canary customer" placing orders continuously).

### Failure Modes

| Failure | Handling |
|---------|---------|
| Recommendations / reviews service down | PDP renders without that widget (timeouts, fallbacks) |
| Search cluster degraded | Serve cached results, disable re-ranker, fall back to simpler ranking |
| Inventory service slow at checkout | Short timeout; fail checkout closed (no oversell), show retry |
| Payment provider outage | Route to secondary PSP; queue orders in `PAYMENT_PENDING`; retry; backoff |
| Duplicate order submit (double-click) | Idempotency key + unique constraint |
| Reservation expires while paying | Re-attempt reserve; if stock gone, void auth and notify |
| Order saga crash | Persisted state + recovery worker; idempotent steps |
| Kafka lag → stale search/price | Show "price at checkout" notice; re-validate on cart; lag alerts |
| Hot partition (viral SKU) | Sharded counters, Redis gate, cache, single-writer queue |
| Warehouse stock mismatch (phantom inventory) | Cycle counts, reconciliation, pick-fail feedback decrements available, auto-cancel + apology credit |
| Region outage | Re-home traffic; replay events; accept brief read staleness |
| Cart service partition | Dynamo-style multi-version merge; always accept writes |

### Common Mistakes

| Mistake | Fix |
|---------|-----|
| One shared relational DB for everything | Service-per-domain with its own store; events for integration |
| Reserving inventory at add-to-cart | Reserve at checkout with TTL |
| `SELECT stock; if > 0 UPDATE` | Single atomic conditional update (or Lua / Dynamo condition) |
| Trusting search/PDP price at payment | Binding quote re-priced at checkout, stored as snapshot |
| Strong consistency for reviews, search | Accept eventual; CQRS projections |
| Querying the Order DB for history lists | Denormalised order-history read model |
| Charging at order time | Authorise at order, capture at shipment |
| Ignoring flash-sale hot keys | Sharded counters / Redis gate / queue / waiting room |
| Making all PDP dependencies critical | Optional widgets with timeouts and fallbacks |
| No idempotency on order creation | Idempotency-Key + dedupe store |

> [!NOTE]
> The simplest mental model: **catalog and discovery are AP and cached to death; checkout is CP and guarded by atomic inventory operations and an idempotent saga.** Spend your time on the boundary between the two.

### Interview Follow-ups

- *"How do you prevent overselling the last item?"* — Atomic conditional update on `(sku, warehouse)`: `UPDATE ... SET reserved = reserved + q WHERE on_hand - reserved >= q`; zero rows affected = reject. No read-then-write. Equivalent: DynamoDB condition expression or a Redis Lua script.
- *"Why not reserve stock when the item is added to the cart?"* — Abandoned carts (90%+) would lock stock; conversion is low. Reserve at checkout with a 10–15 min TTL.
- *"What if payment takes longer than the reservation TTL?"* — Extend TTL when a payment attempt is in flight (reservation heartbeat), or re-reserve after payment; if stock is gone, void the auth and notify. Hold during 3DS redirects.
- *"How do you handle flash sales on a single SKU?"* — Redis atomic gate for admission, sharded inventory counters or a single-writer queue per SKU, CDN-cached static page, waiting room, per-customer quantity limits, bot protection.
- *"How do you keep the cart always available?"* — Dynamo-style store (DynamoDB/Cassandra), writes accepted in any AZ/region, conflicts resolved by merge (union), Redis as an accelerator not a source of truth.
- *"How do you merge guest cart with user cart on login?"* — Union by SKU; for duplicates take max or sum (policy); item-level timestamps/tombstones for removals; clear the guest cart after merge; make the operation idempotent.
- *"Where do you store carts — Redis or a database?"* — Durable store (DynamoDB) with Redis cache. Redis-only risks losing carts on failover; carts are revenue-critical but not money-critical, so eventual consistency is acceptable.
- *"How does search stay in sync with the catalog?"* — CDC/event from catalog → Kafka → indexer; idempotent upserts with version numbers; periodic full reindex into a new index behind an alias for schema changes and drift repair.
- *"Search shows 'in stock' but checkout fails — acceptable?"* — Yes, eventually consistent display; authoritative check at reservation. Reduce frequency by pushing stock events to the index and showing "low stock" thresholds rather than exact counts.
- *"How do you rank search results?"* — Retrieval with BM25 + filters, learning-to-rank on conversion/click/sales features, personalisation, availability and delivery speed boosts, sponsored slots; evaluate by A/B tests on conversion and revenue.
- *"How do you do faceted navigation efficiently?"* — Elasticsearch aggregations with cached facet results for popular queries; pre-compute category-level facets; limit facet cardinality.
- *"How do you handle price changes after the user added to cart?"* — Show updated price with a notice; checkout always re-prices; the order stores the snapshot used at confirmation.
- *"How do you implement coupons without abuse?"* — Eligibility rules, per-user and global usage limits via atomic counters, stackability rules, reservation at checkout and release on failure, anomaly detection for coupon farming.
- *"How do you design the order saga and make it reliable?"* — Orchestrator with persisted state, idempotent steps, compensations, transactional outbox, timeouts, recovery worker, dead-letter queue and manual-review tooling.
- *"Why capture payment at shipment instead of order time?"* — Cancellations before shipping are common; auth-hold avoids refund costs; some regulations restrict charging before dispatch; split shipments are charged separately.
- *"How do you split an order across warehouses?"* — Fulfilment optimiser picks nodes by stock, distance, cost, and promise date; prefers single shipment but accepts splits when it saves days; each shipment has its own tracking and capture.
- *"How do you show an accurate delivery date?"* — Compute at PDP/checkout from the nearest stocked node, cut-off times, carrier transit tables, and capacity; update on exceptions; track promise-vs-actual to tune.
- *"How would you design order history so it's fast?"* — CQRS read model keyed by `user_id` sorted by date; built from order and shipment events; includes product snapshots; paginated with cursors; archival of old orders to cold storage.
- *"I placed an order and don't see it — what happened?"* — Read-your-writes gap on the projection. Fix by writing to the view synchronously, returning order data from the POST, or reading from the primary shortly after writes.
- *"How do you prevent duplicate orders from double-clicks or retries?"* — Idempotency key per checkout attempt, unique constraint `(user_id, idempotency_key)`, return the stored response on replay; disable the button client-side as UX only.
- *"How do you scale reviews?"* — Partition by product; precomputed aggregates; async moderation; ES for search; top-reviews cache; partitions for products with millions of reviews; spam detection pipeline.
- *"How do you prevent fake reviews?"* — Verified-purchase gating, behavioural/graph anomaly detection, text models, rate limits, delay publication on suspicious accounts, seller-linked clusters.
- *"How do recommendations work at Amazon-scale?"* — Item-to-item CF precomputed offline for every item, session signals for freshness, ANN retrieval + ranker, KV serving with fallbacks; measure CTR, conversion, revenue lift.
- *"How do you avoid the thundering herd on a popular PDP?"* — CDN + request coalescing (single flight), stale-while-revalidate, jittered TTLs, local in-process caches.
- *"What database for inventory — SQL or NoSQL?"* — Whichever gives single-row atomic conditional updates at scale: sharded Postgres/MySQL with row locks, or DynamoDB conditional writes. The key is partitioning by SKU and keeping each reservation a single-item atomic operation.
- *"How do you reconcile inventory with the warehouse?"* — WMS events (receipts, picks, adjustments) update on-hand; periodic cycle counts; discrepancies create adjustment ledger entries; pick failures reduce availability immediately and trigger re-allocation.
- *"How do you handle returns and refunds?"* — Return request → RMA label → inbound scan → inspection → restock (or write-off) → refund via Payment service; inventory re-enters as sellable only after inspection.
- *"How is multi-seller marketplace inventory different?"* — Seller feeds can be stale, so apply safety buffers and cancel-rate penalties; FBA inventory is trusted (Amazon-held); the buy-box algorithm favours reliable offers.
- *"How do you handle taxes and currencies?"* — Dedicated tax service (jurisdiction rules, GST/VAT) invoked at checkout; price lists per marketplace; store currency alongside amounts, use minor units/decimals not floats; FX only for reporting.
- *"How do you protect checkout under load?"* — Priority load shedding, queues in front of order creation, circuit breakers on dependencies, bulkheads per dependency, autoscaling with warm pools, waiting room for events.
- *"Multi-region: how do you avoid conflicts on orders?"* — Home each customer's orders to one region; replicate asynchronously for DR; global tables with last-writer-wins only for non-critical data (preferences, wish lists).
- *"How do you roll out schema changes in the catalog safely?"* — Backward-compatible additive changes, dual-write/dual-read, versioned events, new index behind alias, backfill jobs, then cut over and deprecate.
- *"How do you ensure the system is cost-efficient?"* — Cache hit ratios, tiered storage (hot DynamoDB → S3), on-demand vs provisioned capacity, spot instances for batch, CDN offload, compress and right-size images, ES hot/warm tiers.
- *"What metrics tell you the platform is healthy?"* — Conversion funnel, p99 latencies per page, checkout and payment success, oversell count, order backlog, cart-write errors, search zero-results, shipment exception rate.
- *"How would you evolve this from a monolith?"* — Start with modular monolith (catalog, cart, order, payment modules); extract search first (different scaling), then inventory and checkout; adopt events/outbox and CQRS where read/write scaling diverges.
