# Q: Design BookMyShow / a ticket booking system (Ticketmaster).

**Answer:**

Looks like CRUD plus payments. It is really a **contention problem**: thousands of people want the *same few seats* at the *same instant*, and the system must never sell a seat twice, never lose a paid seat, and never let a held-but-abandoned seat stay locked forever. Everything else (search, catalogue, notifications) is easy and read-heavy.

### Requirements

**Functional**:
- Browse cities, movies/events, theatres/venues, shows (showtimes).
- Search and filter (city, date, language, genre, format, price).
- View a **seat map** with live availability and prices per category.
- Select seats (or request "N best seats"), **hold** them for a few minutes, pay, get a confirmed ticket (QR code, email/SMS/push).
- Cancel / refund according to policy.
- Admin / partner: create shows, set pricing, block seats (house seats, distancing, camera positions).

**Non-functional**:
- **Zero double-booking** (hard correctness, strong consistency per seat).
- Seat map load p99 < 300 ms; hold p99 < 200 ms.
- Browsing is eventually consistent (stale by a few seconds is fine); booking is strongly consistent.
- Survive flash sales: 1M+ users hitting one show in the first minute (Taylor Swift / IPL final / Avengers opening).
- Holds expire automatically (typically 5-10 minutes, 2-3 min for flash sales).
- Payment gateways are slow, flaky and call back late; the design must tolerate that.
- Availability: browsing 99.99%; booking can degrade (queue users) rather than fail.

### Capacity Estimation

```
Catalogue:
  500 cities, 10k screens (venues x halls), ~5 shows/screen/day
  = 50k shows/day, ~150 seats avg -> 7.5M seat rows/day
  Keep 60 days forward: ~450M seat rows (tiny: ~100 B each = 45 GB)

Traffic (normal day):
  10M DAU browsing, 1M bookings/day -> ~12 bookings/s avg, ~150/s peak (evening)
  Seat-map reads: 20 per booking -> ~3k/s peak, cacheable

Flash sale (single hot show):
  1M concurrent users, 50k seats, sells out in 60-90 s
  Hold attempts: ~200k-500k req/s at the edge for a few seconds
  Actual successful holds: 50k (bounded by inventory!)
```

Key insight: **write volume is bounded by inventory**, not by users. 1M users fighting for 50k seats means at most 50k successful holds plus a long tail of rejections. Rejections must be *cheap* (served from cache / Redis / edge, never touching the DB).

### Core Entities

```
City ──< Venue ──< Screen ──< Seat (static layout)
Movie/Event ──< Show (screen + start_time + pricing)
Show ──< ShowSeat (seat state PER SHOW)  <-- the contended table
Booking ──< BookingSeat ──> ShowSeat
Payment (1:N per booking: retries)
```

The crucial modeling decision: **separate static seat layout from per-show seat state.** `Seat` = row/number/category, never changes. `ShowSeat` = (show_id, seat_id, status, held_by, hold_expires_at, booking_id, price). One row per seat per show.

### API Design

```
GET  /v1/cities/{city}/shows?movie=&date=&lang=&format=     -> list (cached / ES)
GET  /v1/shows/{show_id}/seatmap                            -> layout + status bitmap + prices
POST /v1/shows/{show_id}/holds
     headers: Idempotency-Key
     body: { seat_ids: ["A5","A6"] } | { count: 3, category: "GOLD", adjacent: true }
     resp: 201 { hold_id, seats, expires_at, price_breakdown } | 409 { unavailable: ["A6"] }
DELETE /v1/holds/{hold_id}                                  -> release early
POST /v1/bookings
     headers: Idempotency-Key
     body: { hold_id, payment_method }
     resp: 202 { booking_id, status: "PAYMENT_PENDING", payment_url / client_secret }
GET  /v1/bookings/{id}                                      -> poll status / ticket
POST /v1/payments/webhook/{provider}                        -> callback (signed, idempotent)
POST /v1/bookings/{id}/cancel                               -> refund per policy
```

Notes:
- Hold and booking are separate resources: **hold = reservation of inventory, booking = commitment of money**.
- `409` returns *which* seats failed so the UI can recolor immediately.
- `202` on booking: confirmation is asynchronous (webhook), the client polls or subscribes via SSE/WebSocket.

### Data Model

```sql
CREATE TABLE show (
    show_id      BIGINT PRIMARY KEY,
    screen_id    BIGINT NOT NULL,
    movie_id     BIGINT NOT NULL,
    starts_at    TIMESTAMPTZ NOT NULL,
    city_id      INT NOT NULL,
    status       TEXT NOT NULL            -- SCHEDULED, ON_SALE, CLOSED, CANCELLED
);

CREATE TABLE show_seat (
    show_id          BIGINT NOT NULL,
    seat_id          INT    NOT NULL,
    category         TEXT   NOT NULL,     -- SILVER/GOLD/RECLINER
    price_cents      INT    NOT NULL,
    status           TEXT   NOT NULL,     -- AVAILABLE | HELD | BOOKED | BLOCKED
    hold_id          UUID,
    hold_expires_at  TIMESTAMPTZ,
    booking_id       BIGINT,
    version          INT    NOT NULL DEFAULT 0,   -- optimistic lock
    PRIMARY KEY (show_id, seat_id)
);
CREATE INDEX ON show_seat (show_id, status);
CREATE INDEX ON show_seat (hold_expires_at) WHERE status = 'HELD';

CREATE TABLE booking (
    booking_id    BIGINT PRIMARY KEY,
    user_id       BIGINT NOT NULL,
    show_id       BIGINT NOT NULL,
    hold_id       UUID   NOT NULL UNIQUE,         -- one booking per hold
    status        TEXT   NOT NULL,                -- PENDING_PAYMENT, CONFIRMED, FAILED, EXPIRED, CANCELLED, REFUNDED
    amount_cents  INT    NOT NULL,
    idem_key      TEXT   NOT NULL,
    created_at    TIMESTAMPTZ DEFAULT now(),
    UNIQUE (user_id, idem_key)
);

CREATE TABLE booking_seat (
    booking_id BIGINT NOT NULL,
    show_id    BIGINT NOT NULL,
    seat_id    INT    NOT NULL,
    PRIMARY KEY (booking_id, seat_id),
    UNIQUE (show_id, seat_id)         -- THE last line of defence against double-booking
);

CREATE TABLE payment (
    payment_id    BIGINT PRIMARY KEY,
    booking_id    BIGINT NOT NULL,
    provider      TEXT, provider_txn_id TEXT UNIQUE,
    status        TEXT,               -- INITIATED, AUTHORIZED, CAPTURED, FAILED, REFUNDED
    amount_cents  INT
);
```

`UNIQUE (show_id, seat_id)` on `booking_seat` means that even if every layer above (Redis, app logic, cache) has a bug, the database physically cannot record two confirmed bookings for one seat.

### High-Level Architecture

```
 Users ──► CDN / Edge (WAF, bot defense, virtual waiting room)
              │
              ▼
        API Gateway ── auth, rate limit, idempotency
         │      │       │         │            │
         ▼      ▼       ▼         ▼            ▼
   Catalogue  Search  Seatmap   Booking     Payment
   Service    (ES)    Service   Service     Service ◄──► PSP (Razorpay/Stripe)
      │         ▲       │        │  │           │            │
      ▼         │       ▼        │  │           ▼            │ webhook
  Postgres   CDC/Kafka  Redis    │  └────► Kafka (events) ◄───┘
  (catalogue)           (seat    ▼              │
                        bitmaps, Postgres       ├──► Ticket/QR service ► SMS/Email/Push
                        holds)   (bookings,     ├──► Notification
                                  show_seat)    ├──► Analytics / fraud
                                                └──► Hold-expiry worker
```

Splits:
- **Read path** (catalogue, search, seat map): cached, replicated, eventually consistent.
- **Write path** (hold, book, pay): strongly consistent, single-writer per show.

### The Core Problem: Preventing Double Booking

Three levels, each a defence layer. Interviewers love "what if Redis lies?", so always show the DB backstop.

**Level 1: Naive (wrong).**

```
seats = SELECT * FROM show_seat WHERE show_id=? AND seat_id IN (...) AND status='AVAILABLE'
if len(seats) == n:
    UPDATE show_seat SET status='BOOKED' ...
```

Classic check-then-act race: two transactions both read AVAILABLE, both write BOOKED. Last write wins, two people hold a "confirmed" ticket.

**Level 2: Pessimistic lock.**

```sql
BEGIN;
SELECT * FROM show_seat
 WHERE show_id = ? AND seat_id IN (?, ?)
 ORDER BY seat_id            -- consistent order avoids deadlocks
 FOR UPDATE;
-- check all AVAILABLE (or HELD with expired hold)
UPDATE show_seat SET status='HELD', hold_id=?, hold_expires_at=now()+interval '8 min'
 WHERE show_id=? AND seat_id IN (...);
COMMIT;
```

Correct, simple, but row locks queue every contender on hot seats and hold connections open. Fine for normal shows; collapses under flash load (connection pool exhaustion).

**Level 3: Optimistic / conditional update (single atomic statement).**

```sql
UPDATE show_seat
   SET status='HELD', hold_id=:h, hold_expires_at=now()+interval '8 minutes', version=version+1
 WHERE show_id=:s AND seat_id = ANY(:seats)
   AND (status='AVAILABLE' OR (status='HELD' AND hold_expires_at < now()));
-- rows_affected must equal len(seats), else ROLLBACK and return 409
```

One round trip, no explicit lock held across network calls, compare-and-set semantic. Loser gets 0 rows (or fewer than N) and fails fast. This is the production default for the database layer.

**Level 4: Redis in front for speed (hot shows).**

```
# per-seat key, atomic claim with TTL
SET hold:{show}:{seat} {hold_id} NX EX 480

# multi-seat all-or-nothing: Lua script
for each seat: if redis.call('EXISTS', key) == 1 then return {0, seat} end
for each seat: redis.call('SET', key, hold_id, 'EX', ttl)
return {1}
```

- `SET NX EX` is atomic: exactly one caller wins, and the key evaporates by itself on TTL (no cleanup job needed for the *hold*).
- Lua makes the multi-seat claim atomic (all or none). With Redis Cluster, force all seats of a show onto one slot with hash tags: `hold:{show_123}:A5`.
- Redis is the **fast gate**, Postgres is the **source of truth**. After the Redis claim succeeds, write the HELD row (and later BOOKED row) in Postgres. If Postgres rejects (unique/CAS), release the Redis key.

```
hold(seats):
  1. Lua: claim all seats in Redis           -> fail fast with 409 (no DB hit)
  2. DB: conditional UPDATE to HELD          -> truth
  3. on DB failure: release Redis keys
  4. return hold_id, expires_at
```

> [!NOTE]
> Redis can lose a key (failover, eviction, restart). That is *safe by construction* only because confirm-time checks the DB: the confirm transaction re-verifies `hold_id` matches and the `UNIQUE(show_id, seat_id)` constraint in `booking_seat` guards the final write. Redis accelerates; it never decides alone.

### Isolation Levels

| Level | Behaviour for seat booking |
|-------|---------------------------|
| Read Committed (Postgres default) | Safe **if** you use conditional `UPDATE ... WHERE status='AVAILABLE'` or `FOR UPDATE`. Plain read-then-write is not safe. |
| Repeatable Read / Snapshot | Detects write-write conflict and aborts the loser with serialization error; app must retry. Good with optimistic style. |
| Serializable (SSI) | Strongest; handles "N seats adjacent" read-sets without manual locks; abort rate rises under contention, needs retry loop. |
| MySQL InnoDB RR | Gap/next-key locks on range scans can deadlock on `IN (...)`; lock rows in sorted order. |

Interview rule of thumb: do not rely on a high isolation level to fix a logic bug. **Make the invariant a constraint** (unique index, conditional update) and keep isolation modest.

### Hold Expiry (TTL)

Holds must disappear if the user abandons the page.

**Approach A: lazy expiry (recommended).** A seat is "available" if `status='AVAILABLE' OR (status='HELD' AND hold_expires_at < now())`. Every claim query already encodes this. No race with a cleanup job; correctness never depends on a worker running on time.

**Approach B: Redis TTL.** The key expires natively; seat map reads treat "no key" as available.

**Approach C: sweeper worker.** Periodically `UPDATE show_seat SET status='AVAILABLE' WHERE status='HELD' AND hold_expires_at < now()` (uses the partial index). Purely to keep the seat-map status column tidy and to emit "seat freed" events for the cache and websockets.

Use A + B for correctness, C for hygiene. Avoid delayed-queue-per-hold (millions of timers; SQS/Kafka delay semantics are loose).

**Hold TTL tuning**: 5-10 minutes normal; 2-3 minutes during flash sales (inventory is scarce, shorten idle locking). Show a visible countdown. Extend the TTL once (+2 min) when the user enters the payment page, to avoid expiring mid-3DS.

### Payment Callback vs Hold Expiry Race

The nastiest scenario:

```
t=0     user holds seats (expires t=480)
t=470   user pays; PSP processing slow
t=480   hold expires; seats freed
t=485   another user holds & pays for the same seats
t=490   first PSP webhook: payment SUCCESS
```

Money was taken for seats now owned by someone else. Defences, layered:

1. **Extend hold when payment starts.** On `POST /bookings`, set `hold_expires_at = max(existing, now() + payment_window)` (e.g. 10 min) and mark `PAYMENT_PENDING`. A seat in `PAYMENT_PENDING` is not claimable by anyone else, even after the original TTL.
2. **Confirm transaction on webhook:**
   ```sql
   BEGIN;
   SELECT * FROM booking WHERE booking_id=? FOR UPDATE;   -- serialize with expiry job
   -- if status = CONFIRMED -> no-op (idempotent)
   -- if status = PENDING_PAYMENT and all show_seat rows still hold_id = booking.hold_id:
   UPDATE show_seat SET status='BOOKED', booking_id=? WHERE ... AND hold_id=?;   -- must hit N rows
   INSERT INTO booking_seat ...;   -- UNIQUE(show_id, seat_id) backstop
   UPDATE booking SET status='CONFIRMED';
   COMMIT;
   ```
3. **If confirm fails** (seats lost): auto-**refund** the capture (or void the authorization) and notify the user; optionally offer alternative seats. This is the compensating transaction, and it must exist even if rare.
4. **Prefer authorize-then-capture:** authorize at payment time, capture only after the seat is confirmed. If seats are lost, just void the auth: no money moves, no refund fees.
5. **Verify with the PSP**, don't trust the browser redirect. The webhook plus a reconciliation job (`GET payment status` for every `PAYMENT_PENDING` older than N minutes) cover lost webhooks.

Expiry job and webhook both take `FOR UPDATE` on the booking row, so they serialize: either the expiry job moves it to `EXPIRED` first (webhook then refunds) or the webhook confirms first (expiry job sees `CONFIRMED` and skips).

### Idempotency

Retries happen everywhere: mobile clients on bad networks, gateway timeouts, webhook redelivery.

- **Client to API**: `Idempotency-Key` header; stored in `booking(user_id, idem_key) UNIQUE`. A replay returns the original response, not a second booking.
- **Hold** request idempotent by `(user_id, show_id, idem_key)`: re-sending returns the existing hold.
- **Webhook**: dedupe on `provider_txn_id UNIQUE`; the confirm transaction is a state-machine transition guarded by current state, so replays are no-ops.
- **Notifications**: dedupe key `booking_id + channel`, so the user doesn't get 3 SMS.
- **Outbox pattern**: write the `BookingConfirmed` event in the same DB transaction (outbox table), a relay publishes to Kafka. No lost or phantom events.

### Booking State Machine

```
        hold ok                 pay clicked            webhook SUCCESS + seats ok
(none) ─────────► HELD ─────────────────► PAYMENT_PENDING ───────────────► CONFIRMED
                   │                            │   │                          │
        TTL/cancel │                            │   └─ webhook FAIL ► FAILED   │ cancel
                   ▼                            │ timeout (no webhook,         ▼
                EXPIRED ◄───────────────────────┘  PSP says failed)        CANCELLED ─► REFUNDED
                                                SUCCESS but seats gone ─► REFUND_REQUIRED ─► REFUNDED
```

Every transition is a guarded `UPDATE ... WHERE status = <expected>`; zero rows affected means someone else moved it first, which is a safe no-op or an alert.

### Seat Map Caching and Delivery

The seat map is the hottest read. 1M users refresh it every couple of seconds in a flash sale.

- **Split static from dynamic.** Layout (coordinates, categories, prices) is immutable per screen layout: cache at CDN forever (versioned URL). Availability is a **bitmap**: 1 bit or 2 bits per seat (available / held / booked / blocked). A 2000-seat hall = ~500 bytes.
- **Dynamic status in Redis** as a bitfield or hash per show; served by Seatmap Service, cached at edge for **1-2 seconds** (micro-caching).
- **Push deltas**: when a seat changes, publish `{show_id, seat_id, status}` to Kafka; a WebSocket/SSE fan-out service pushes to viewers of that show so seats turn grey in near-real time without polling. Only subscribe users who have the seat map open; coalesce updates per 500 ms.
- **Stale reads are fine** because the hold request is authoritative. Showing a seat as free that was just taken just yields a clean 409 and the UI refreshes. Never trust the displayed map.
- **Cache invalidation**: write-through on every status change (the same code path that updates Redis state updates the bitmap), plus TTL as a safety net.

### Search and Browse (Elasticsearch)

- Documents: one per show (denormalized: movie title, language, genres, format, city, venue name, geo point, start time, min price, availability flag).
- Fed by **CDC** (Debezium) from catalogue Postgres into Kafka, then to Elasticsearch. Indexing lag of seconds is fine.
- Queries: filter by city + date range + language + format, full text on movie/venue name, geo-distance for "theatres near me", sorted by popularity/time.
- Availability in search is **approximate** ("Filling fast", "Sold out" from a rolling counter), not a per-seat query. Authoritative availability only on the seat map.
- Typeahead via ES `completion` suggester or edge n-gram analyzer.
- Cache popular listing queries (city + date) in Redis/CDN for 30-60 s: browse traffic is highly repetitive.

### Sharding and Scale

- **Shard bookings and show_seat by `show_id`** (or `city_id` then show). All contention for one show is on one shard; no cross-shard transactions for booking. A multi-seat booking is always within one show.
- **Hot show problem**: one show on one shard is a hot partition. Mitigations: dedicate resources (own Redis node and DB partition) for flagged "hot events"; the in-memory Redis gate absorbs 99% of rejections; DB sees only ~inventory-sized writes.
- **Catalogue** (cities, venues, movies) is small and read-heavy: replicate everywhere, cache aggressively.
- **Partition by time**: shows in the past are archived (partition drop), keeping `show_seat` small and indexes hot.
- **Per-show single writer** (advanced): route all booking requests of a show to a single owner (consistent hashing; actor model or a Kafka partition keyed by show_id). The owner holds the seat state in memory and serializes. Gives huge throughput, no lock contention, but needs fail-over and persistence (log + snapshot).

### Seat Selection Modes

**Pick specific seats**: seat IDs in request; claimed atomically (all-or-nothing). User sees live map.

**Auto-assign ("best available N seats")**:
- Server picks, so there is no client/server race on stale maps, and fewer conflicts.
- Algorithm: for category, scan rows from best (centre, middle rows) outward for **N contiguous available seats**; with a bitmap per row this is a bit-run search (`x & (x>>1) & ... `), O(rows).
- Score candidates: distance from screen centre, same row, no orphan single seat left.
- Try claim; on conflict, take the next candidate. Retry loop with bound (3-5), then fall back to "split across rows" or fail.

**Multi-seat adjacency**:
- Maintain per-row bitmaps in Redis; adjacency search is bit arithmetic.
- **Avoid orphan seats** (single empty seat between bookings), commonly a business rule: reject selections that leave exactly one gap, or pre-block per policy.
- Social distancing / blocked seats are just `BLOCKED` rows.
- Group of 6+: allow multi-row blocks; require seats in a single category.

### Pricing

- Static base price per category per show plus rules (weekend, premium format, early bird).
- **Dynamic pricing** (events/concerts): demand-based tiers, e.g. platinum tickets; Ticketmaster-style. Price is **locked at hold time** and stored with the hold; the user pays what they saw.
- Taxes, convenience fee, GST computed server-side; return a price breakdown with the hold.
- Offers/coupons validated at hold or checkout; coupon redemption counts need atomic increments (Redis DECR or DB row) to avoid overuse.

### Flash Sales and the Virtual Waiting Room

When 1M users arrive for 50k seats, scaling servers to absorb 1M req/s is wasteful and still gives a bad UX (random winners, refresh storms). Use **admission control**:

```
User arrives ──► Edge (CDN worker) ──► Queue service
                                         │ assigns position, signed token
                                         ▼
                           Rate-controlled admission (e.g. 2,000 users/10 s,
                           tuned to booking-service capacity and remaining inventory)
                                         │ admitted token (JWT, short TTL)
                                         ▼
                                Booking flow (seatmap, hold, pay)
```

- **Queue position**: randomised for fairness (lottery among everyone who arrives in the first N minutes) rather than strict FIFO, which rewards bots and fast connections.
- **Admission rate** is driven by feedback: hold success rate, DB latency, remaining seats. Admit more when seats go unsold (holds expiring), pause when saturated.
- **Token**: signed, bound to user/device/IP-ish, expires; the booking API rejects requests without it (checked at the edge, cheap).
- **Static waiting page** served from CDN; polling replaced by SSE/long-poll with jitter. The room itself must be nearly stateless: Redis sorted set or even stateless HMAC "ticket numbers" with a moving admission watermark.
- **Pre-registration / verified fan / presale codes** (Ticketmaster): only registered, verified accounts receive entry; cuts bots dramatically.
- **Inventory-aware**: when remaining < admitted users, show "sold out" at the edge from a flag in the CDN/KV, so the late arrivals never touch origin.
- **Pre-warm** caches, connection pools, DB, autoscaling before onsale time; scheduled scale-up not reactive.
- **Per-user limits**: max 4-6 tickets per account/payment card per show; one active hold per user per show.

### Bots, Scalpers and Abuse (Ticketmaster-style)

| Threat | Defence |
|--------|---------|
| Scripted seat grabbing | Edge bot detection (device fingerprint, behaviour, TLS fingerprint), CAPTCHA/proof-of-work at queue entry, not at every click |
| Mass fake accounts | Phone/KYC verification, account age requirements for hot onsales, verified-fan registration |
| Seat hoarding (hold and never pay) | Short hold TTL, one hold per user, penalty (cool-down, lower priority) for repeated expiring holds, per-user hold budget |
| Resale scalping | Purchase limits, name-bound or dynamic-QR tickets (rotating barcode), controlled transfer/resale platform with price cap |
| Card testing / stolen cards | PSP risk scoring, 3DS/OTP, velocity checks by card/IP/device, Radar-like ML, delayed ticket delivery on high-risk orders |
| Distributed residential-proxy bots | Behavioural + reputation signals, rate limits per account and per payment instrument, waiting-room randomisation (speed is no advantage) |
| Insider/API abuse | Hold and booking endpoints only reachable with queue-admission token; signed requests; per-token rate limits |

Keep holds cheap for legitimate users and costly for abusers; do not punish a user for slow payment on a flaky bank page (extend on 3DS).

### Refunds and Cancellations

- Policy engine: time-to-show buckets (free cancel up to 4 h before, partial after), per-event rules (concerts usually no refund; movies often convenience fee non-refundable).
- Cancellation flow: guarded transition `CONFIRMED -> CANCELLED`, release seats (`BOOKED -> AVAILABLE`, delete `booking_seat`), enqueue refund, emit event so the seat map updates.
- **Refund is async and idempotent**: refund request id = `booking_id + attempt`; reconcile with PSP; to original source, or wallet credit instantly (cheaper, retains money).
- **Show cancelled by the venue**: bulk job, per-booking idempotent refunds, rate limited to the PSP, status page and notifications.
- Partial cancellation (one ticket of a group): handle by `booking_seat` row removal plus pro-rata refund; keep immutable ledger entries.
- **Double-entry ledger** for money movement; never mutate amounts, append entries.

### Ticket Issuance and Entry

- On `CONFIRMED`, async worker generates the ticket: unique code, QR (signed payload `booking_id|show|seat|nonce|sig`), PDF/wallet pass.
- Entry scanners verify the signature offline and mark used; use a bloom/cache of used ticket IDs to catch duplicates quickly; sync to the central DB (eventually) with conflict rule "first scan wins".
- Rotating QR (refresh every 30 s in the app) kills screenshot reselling.

### High-Level Booking Flow (Sequence)

```
Client          Booking Svc        Redis           Postgres         PSP
  │ POST /holds     │                │                │              │
  ├────────────────►│ Lua claim      │                │              │
  │                 ├───────────────►│                │              │
  │                 │ ok             │                │              │
  │                 │◄───────────────┤ CAS UPDATE HELD│              │
  │                 ├───────────────────────────────►│              │
  │ 201 hold_id,ttl │◄───────────────────────────────┤              │
  │◄────────────────┤                │                │              │
  │ POST /bookings  │ extend TTL, create booking PENDING_PAYMENT      │
  ├────────────────►├───────────────────────────────►│ create intent │
  │                 ├────────────────────────────────────────────────►│
  │ 202 + pay link  │                │                │              │
  │◄────────────────┤                │                │   user pays   │
  │                 │                │                │ webhook      │
  │                 │◄───────────────────────────────────────────────┤
  │                 │ confirm txn (FOR UPDATE, BOOKED, outbox)        │
  │                 ├───────────────────────────────►│              │
  │ push/poll: CONFIRMED + ticket    │                │              │
```

### Multi-Region and Availability

- **Single-writer per show**: each show's booking state lives in one region (home region by venue city). Avoid multi-master on seats; conflict resolution on a seat is impossible without coordination.
- Browse/search/catalogue replicated read-only to all regions; stale okay.
- DR: cross-region async replica; on region failure, promote with RPO of seconds. Reconcile in-flight holds (they expire anyway) and payments via PSP reconciliation.
- Redis failure: fall back to DB-only claim path (slower, still correct); waiting room lowers admission rate automatically.
- PSP failure: multiple PSPs with routing and failover; holds extended while payment retries; user sees "try another method".
- Degradation ladder: shed browse (serve cached), then limit admission, never accept bookings that cannot be made safe.

### Observability

- **Business SLIs**: hold success rate, hold-to-booking conversion, payment success rate, seats stuck in `PAYMENT_PENDING`, refunds-due-to-lost-seat (should be near 0), oversell count (must be 0, alert on any).
- Traces: gateway -> hold -> payment -> webhook; correlate by `booking_id`.
- Invariant checker job: `SELECT show_id, seat_id, count(*) FROM booking_seat GROUP BY 1,2 HAVING count(*) > 1` (impossible with the unique index, but detect schema drift), and compare `show_seat.status='BOOKED'` against `booking_seat` rows.
- Waiting room dashboards: queue depth, admit rate, abandonment; per-region latency.
- Synthetic bookings (test show) every minute in prod for end-to-end signal.

### Testing

- **Concurrency tests**: N=1,000 threads claiming the same seat; assert exactly one success. Jepsen-style fault injection for Redis failover mid-hold.
- **Property tests** on the state machine: no illegal transitions; every seat is in exactly one state.
- **Race harness**: deterministic ordering of webhook vs expiry with controllable clocks.
- **Load tests**: replay flash-sale shapes (spike, not ramp) incl. waiting room and bots; game days before major onsales.
- **Chaos**: kill PSP responses, delay webhooks 10 minutes, duplicate webhooks, clock skew between app servers (use DB time `now()` for expiry decisions, not app time).

### Storage Choices

| Data | Store | Why |
|------|-------|-----|
| Bookings, payments, show_seat | PostgreSQL / MySQL (sharded by show) | ACID, unique constraints, CAS |
| Live seat holds / seat bitmap | Redis (cluster, hash-tagged per show) | Atomic `SET NX EX`, Lua, sub-ms |
| Catalogue (cities, venues, movies) | Postgres + read replicas + cache | Small, relational, read-heavy |
| Search | Elasticsearch / OpenSearch | Full text, geo, facets |
| Seat layouts, posters, ticket PDFs | S3 + CDN | Immutable assets |
| Events (booking, payment) | Kafka | Decoupling, replay, outbox relay |
| Waiting room state | Redis + edge KV | Fast, ephemeral |
| Analytics | ClickHouse / BigQuery | Funnel, pricing, fraud |
| Ledger | Append-only table (Postgres) | Auditable money movement |

### Failure Modes

| Failure | Handling |
|---------|----------|
| Two users click the same seat simultaneously | Redis `SET NX` / DB conditional UPDATE; loser gets 409 |
| Redis loses hold keys (failover) | DB conditional UPDATE still guards; confirm re-verifies `hold_id`; unique index backstop |
| Payment succeeds after hold expiry | Seats `PAYMENT_PENDING` locked during payment; else refund/void compensation |
| Webhook lost | Reconciliation job polls PSP for pending payments |
| Webhook duplicated | `provider_txn_id UNIQUE`; state guard makes confirm idempotent |
| Booking service crashes after Redis claim, before DB write | Redis TTL frees seats; no leak |
| Crash after DB `BOOKED` but before event publish | Outbox row in same transaction; relay retries |
| Hot show overwhelms DB | Waiting room + Redis gate + dedicated shard |
| User double-clicks "Pay" | Idempotency key; one booking per `hold_id` unique |
| App server clock skew | Use DB/Redis server time for expiry |
| Elasticsearch behind | Search stale; booking unaffected (separate path) |
| Venue cancels show | Bulk idempotent refunds, notify, seats closed |
| PSP outage | Failover PSP, extend holds, queue retries |

### Common Mistakes

| Mistake | Fix |
|---------|-----|
| Read-then-write seat check | Conditional `UPDATE ... WHERE status='AVAILABLE'` or `SELECT FOR UPDATE` |
| Trusting only Redis (or only app cache) for correctness | DB unique constraint + hold_id check at confirm |
| Locking seats forever until a cleanup cron runs | Lazy expiry in the claim predicate + TTL keys |
| Confirming the booking from the browser redirect | Confirm only from verified PSP webhook / server-side status check |
| No compensation for payment-after-expiry | Void/refund path, plus lock during `PAYMENT_PENDING` |
| Scaling the app tier for a flash sale instead of queueing | Virtual waiting room + admission control |
| Strict FIFO queue | Randomised admission; FIFO favours bots |
| Updating the whole seat map per click via polling | Bitmaps, micro-caching, push deltas |
| Distributed transaction across Redis + DB + PSP | Saga with compensations and idempotent steps |
| Multi-master writes for seats | Single writer per show (home region) |
| Using app-server time for TTLs | DB / Redis server clock |
| Searching availability in Elasticsearch per seat | ES for discovery; authoritative availability from seat service |

> [!NOTE]
> The core of the interview is the **hold lifecycle**: atomic claim, TTL expiry, payment race, and compensation. Say out loud: "Redis for speed, DB constraint for truth, saga with refund for the rare mismatch, waiting room so the system never sees the raw stampede."

### Interview Follow-ups

- *"How do you prevent two users from booking the same seat?"* — Atomic claim: Redis `SET NX EX` (Lua for multi-seat) as a fast gate, then a conditional DB `UPDATE ... WHERE status='AVAILABLE' OR hold expired`; final backstop is `UNIQUE(show_id, seat_id)` on `booking_seat`.
- *"Why not just SELECT FOR UPDATE?"* — It works and is correct, but on hot shows it queues transactions, holds connections, and risks deadlocks (sort seat ids to mitigate). Conditional UPDATE is a single statement and fails fast; Redis absorbs the stampede.
- *"Optimistic vs pessimistic locking here?"* — Contention is extreme on a few rows, so naive optimistic retries waste work; but we do not retry, we simply fail the loser fast (409). Hence CAS on a version/status column is ideal. Pessimistic for admin bulk operations.
- *"What isolation level do you use?"* — Read Committed with conditional updates is enough. Serializable only if you do complex read-set logic like adjacency search in SQL, and then you need a retry loop. Invariants live in constraints, not isolation.
- *"What if Redis crashes and loses all holds?"* — Holds are also in the DB (HELD rows with `hold_expires_at`); recovery rebuilds Redis from DB or runs DB-only mode. Worst case a seat is claimable twice in Redis but the DB CAS rejects the second.
- *"What if the DB write succeeds but the app crashes before replying?"* — Client retries with the same idempotency key; server returns the existing hold. Orphan holds expire via TTL.
- *"User pays but the hold expired, what now?"* — Extend hold on payment start, lock `PAYMENT_PENDING` seats, confirm in a transaction re-checking `hold_id`. If seats are lost anyway: void the authorization or refund, notify, offer alternatives.
- *"Auth/capture or direct charge?"* — Authorize first, capture after confirming seats; void on failure (no fee, instant). Direct capture forces refunds, which cost money and days.
- *"How do you handle webhooks arriving twice or out of order?"* — Dedupe by provider txn id, drive a state machine with guarded transitions; ignore transitions that go backwards; reconciliation job as safety net.
- *"How long should the hold be?"* — 5-10 min normally, 2-3 min for flash sales, extended on payment start (3DS/OTP). Shorter holds = more throughput, worse UX; tune by observed payment durations (p95).
- *"How do you stop people hoarding seats?"* — One active hold per user per show, per-user ticket cap, short TTL, cool-down after repeated expired holds, bot detection, verified accounts.
- *"How does the system survive a 1M-user spike for 50k seats?"* — Virtual waiting room at the edge with randomised, inventory-aware admission; cheap rejections from cache/Redis; pre-warmed capacity; only bounded successful holds reach the DB.
- *"Why randomise the waiting room?"* — FIFO rewards bots and low-latency clients; randomising among early arrivals makes speed irrelevant and is perceived as fairer.
- *"How do you serve the seat map to 1M users?"* — Immutable layout on CDN; compact availability bitmap from Redis micro-cached 1-2 s; push deltas via WebSocket/SSE; stale display is harmless as the hold is authoritative.
- *"Do you show real-time seat colour changes?"* — Yes via pub/sub fan-out per show, coalesced; fall back to polling with jitter. Only for users with that show open.
- *"How do you search shows?"* — Elasticsearch with denormalized show docs fed by CDC; filters (city/date/lang/format), geo-distance, popularity sort; availability approximate. Cache hot queries.
- *"How do you keep ES consistent with the DB?"* — Eventually: CDC via Kafka, idempotent upserts by show id/version; periodic full reindex; source of truth remains Postgres.
- *"How do you shard?"* — By `show_id` (all contention local, no cross-shard transactions); catalogue replicated; hot events pinned to dedicated resources; time partitions for old shows.
- *"How do you handle a single extremely hot show?"* — Dedicated Redis/DB capacity, edge-level sold-out flag, waiting room, per-show single-writer owner, inventory-bounded writes.
- *"Auto-assign vs user-picked seats?"* — Auto-assign reduces conflicts (server chooses with fresh state, retries on conflict) and improves packing; picking needs live map and yields 409s. Offer both; auto-assign for flash sales.
- *"How do you find N adjacent seats efficiently?"* — Row bitmaps and bit-run search; score candidates; avoid orphan seats; bounded retry on conflict.
- *"What about orphan single seats?"* — Business rule: reject selections leaving a single isolated seat, or auto-suggest adjusting by one; checked at hold time.
- *"Group bookings across categories?"* — Treat as one hold with multiple seat sets; the same all-or-nothing claim; price per category summed.
- *"How do you do dynamic pricing without unfairness?"* — Price locked at hold; tiered release; transparent breakdown; guard against price changes between hold and pay.
- *"How do coupons avoid being overused?"* — Atomic counters (Redis `DECR`/DB conditional update) at checkout; release on payment failure; per-user limits.
- *"How do you process refunds?"* — Async idempotent refund requests; policy engine; ledger entries; to source or wallet; bulk with throttling for cancelled shows; reconcile with PSP.
- *"What if a refund fails at the PSP?"* — Retry with backoff using the same refund id, alert after N attempts, manual queue; user sees "refund initiated".
- *"What if the venue cancels the show?"* — Bulk saga per booking, idempotent, rate-limited to PSP; auto-credit wallet option; proactive notifications.
- *"How do you handle bots like Ticketmaster does?"* — Queue-entry challenge, verified-fan presale, device/behaviour fingerprinting, per-account/card limits, rotating QR, ML risk scoring; hold token required.
- *"How do you stop scalpers reselling?"* — Name-bound tickets, rotating barcodes, controlled transfer with price cap, purchase limits, post-sale cancellation of flagged orders.
- *"How do you detect stolen cards?"* — PSP risk signals, 3DS, velocity rules, device linkage, delayed delivery of high-risk tickets, chargeback feedback loop.
- *"How do you do multi-region?"* — Single writer region per show (home city); read replicas and CDN elsewhere; async DR replica; no multi-master seat state.
- *"What if the primary DB region fails mid flash sale?"* — Promote replica (RPO seconds); holds in flight just expire; payments reconciled via PSP; waiting room pauses admission during failover.
- *"How do you ensure events (email/ticket) are not lost?"* — Transactional outbox, at-least-once Kafka consumers, idempotent handlers keyed by `booking_id`.
- *"Saga or 2PC?"* — Saga: hold -> payment -> confirm, with compensations (release, void/refund). 2PC across PSP is impossible; Redis+DB 2PC is unnecessary given idempotent steps.
- *"How do you test for double-booking?"* — Concurrent claim tests (1000 threads, one winner), chaos with Redis failover and delayed webhooks, invariant checker in prod, property tests on the state machine.
- *"How do you monitor oversell?"* — Invariant job comparing `show_seat` and `booking_seat`; alert on any duplicate or orphan; metric `refunds_due_to_lost_seat`.
- *"Clock skew risks?"* — Use DB/Redis server time for expiry decisions; never compare app-server clocks across nodes.
- *"How would you make this cost-efficient?"* — Redis only for hot/live shows (TTL keys for upcoming days only), cold shows from DB; CDN for static; archive old show_seat partitions; autoscale on schedule for known onsales.
- *"How would the design evolve for events with general admission or tiers?"* — Replace seat rows with a counter per tier (`available_count`) using atomic `UPDATE ... SET available = available - n WHERE available >= n`; holds as decrementing reservations with TTL; same flow.
- *"Offline/entry validation at the venue?"* — Signed QR verified offline; local used-ticket cache; sync with central; first scan wins; rotating codes against screenshots.
- *"What happens during a deployment mid-sale?"* — Stateless services, rolling deploy, state in Redis/DB; backward compatible schema; freeze deploys during hot onsales.
- *"How do you handle users with two tabs/devices?"* — One active hold per user per show; a new hold replaces or returns the old; hold bound to session.
- *"GDPR / PII?"* — Minimal PII, tokenized payments (PSP vault), retention policies on bookings, encryption at rest, access audit.
- *"How do you show 'only 3 seats left' honestly?"* — From the Redis counter, rounded; never promise exactness; do not fabricate scarcity.
- *"What would you build first (MVP)?"* — Postgres only: `show_seat` conditional UPDATE, hold TTL via lazy expiry, one PSP with webhook confirm, idempotency keys; add Redis, waiting room, ES as traffic demands.
