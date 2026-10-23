# Q: Design a ride-sharing service (Uber / Lyft).

**Answer:**

Combines real-time geospatial matching, location tracking, trip lifecycle, payments, surge pricing. The interesting parts are **matching** and **geo-indexing at scale**.

### Requirements

**Functional**:
- Rider requests a ride from A to B.
- Match nearby driver, show ETA.
- Real-time location tracking (driver and rider see each other).
- Trip lifecycle: requested → matched → enroute → in-trip → completed.
- Pricing including surge.
- Payments.

**Non-functional**:
- 10M drivers, 100M riders.
- Driver location updates every 4 seconds while online.
- Match in < 5 seconds.
- Geographic correctness; never match an out-of-range driver.

### Capacity Estimation

- 10M drivers online during peak in a region (e.g., NYC: 100k drivers).
- Position updates: 10M × 1 / 4 sec = 2.5M updates/sec globally.
- Trip requests: 1M trips/hour globally → 300/sec peak in busy cities.

Scale is dominated by **location ingest** and **geo-queries**, not trip volume.

### High-Level Architecture

```
Driver app                                      Rider app
   │                                                │
   │ location updates                               │ trip requests
   ▼                                                ▼
Location ingest ──► Kafka ──► Geo Index Service ──► Matching Service
   │                              │                    │
   ▼                              ▼                    ▼
DriverState DB             Geohash / Redis       Trip Service
(DynamoDB)                                            │
                                                      ▼
                                                  Trip DB (Postgres)
                                                      │
                                                      ▼
                                                  Payments / Notifications
```

### Driver Location Tracking

- Driver app sends `(driver_id, lat, lng, heading, ts)` every 4 seconds via WebSocket or HTTP/gRPC.
- Ingest writes to **DriverState** (latest position only, key=driver_id) — fast key-value, DynamoDB or Cassandra.
- Also publishes to Kafka for downstream (analytics, ML, history).

Don't store every position in a transactional DB. Use a hot store for "latest" and a stream for "history."

### Geo-Indexing

Need: given a rider's location, find drivers within ~3 km, sorted by distance.

**Options**:

**1. Geohash buckets.**

Divide world into geohash cells. Bucket key = geohash prefix.

```
Driver at (40.71, -74.01) → geohash "dr5ru..."
Index:  geohash_prefix_5 → set of driver_ids

Rider query at "dr5ru":
  candidates = drivers in cell "dr5ru" + 8 neighboring cells
  filter by exact distance, sort
```

Implementation: Redis sorted set per geohash cell, or specialized service like Tile38, Uber's H3.

**2. H3 (Uber's hexagonal grid).**

Hexagons over Earth at multiple resolutions. Each cell ID is a 64-bit int.

```
cell = h3.geoToH3(lat, lng, resolution=9)   // ~100 m hexagons

drivers indexed by cell. Query: cell + k-ring of neighbors.
```

Hexagons have uniform distance to neighbors (squares/lat-lng don't). Mathematically cleaner for matching.

**3. R-tree / k-d tree.**

For low-volume queries. Postgres `GIST` index on `point` works for small-scale.

For real-time at scale: **H3 + Redis** is the production answer.

### Location Update Flow

```
driver app ──► WS gateway ──► location service:
                                   1. update DriverState (latest)
                                   2. update H3 index:
                                        old_cell.remove(driver_id)
                                        new_cell.add(driver_id)
                                   3. publish to Kafka "driver-pings"
```

`old_cell.remove` only if cell changed — most pings don't change cells.

### Matching Algorithm

```
rider requests ride at (lat, lng):
  candidate_cells = h3.kRing(cell, k=2)        # ~300m radius
  candidates = union(cell.drivers for cell in candidate_cells)
  filter: driver.status == "available"
  filter: driver.capacity satisfies request
  rank by (distance, ETA, driver-rating, surge)
  send offer to top-K candidates

driver accepts:
  atomic state transition: driver "available" → "matched"
  notify rider
```

If no driver accepts within timeout, retry with wider radius.

### Trip Lifecycle State Machine

```
requested → matched → enroute_pickup → arrived → in_trip → completed
                  ↘─ cancelled (by rider/driver/timeout)
```

Each transition is a row update + event publish. Trip data in Postgres (ACID for billing).

### Pricing / Surge

- Base fare + per-km + per-min.
- Surge multiplier per cell, updated by supply/demand:
  - High request rate, low driver density → 1.5×, 2×.
- ML-driven in production; rules-based in interview.
- Lock the price at request time (don't change mid-trip).

### Payments

Separate service:
- Authorize hold at trip start.
- Capture on completion.
- Refund on cancellation policy match.
- Settle to driver weekly.

Idempotency key per trip. Use a payments provider (Stripe / Adyen); don't store cards.

### Real-Time UI Updates

Both apps need to see each other's position during a trip:

- WebSocket from driver → server → rider.
- Server forwards position only to the matched rider (privacy).
- Tail: 5–10 ms latency from driver ping to rider screen.

### Storage Choices

| Data | Store | Why |
|------|-------|-----|
| Driver latest position | DynamoDB / Redis | Key-value, hot writes |
| Geo index | Redis (H3 sets) | Range queries on cells |
| Trips | Postgres | ACID for money |
| Trip history archive | S3 + Athena | Cheap long-term |
| Driver profile | Postgres | Identity, vehicle, docs |
| Location history | Kafka → S3 | Streaming + archive |
| Real-time analytics | ClickHouse / Druid | Live dashboards |

### Failure Modes

| Failure | Handling |
|---------|---------|
| Driver app loses connection mid-trip | Trip continues based on last known state; reconnect catches up |
| Matching service overloaded in a city | Per-region matching cluster; shard by city |
| Payment provider down at trip end | Capture deferred; trip ends; retry payment async |
| Geo index hot cell (peak Times Square) | Shard within cell by driver_id hash |
| Driver disappears off-map | Heartbeat timeout → mark offline; in-trip case → alert ops |

### Multi-Region

- Each city's matching runs in nearest region.
- Driver state and geo index local to region.
- Cross-region trip (very rare) → coordinate via city-of-origin.

### Privacy

- Geofence driver-visible-to-rider window (only during trip).
- Mask phone numbers via proxy (twilio number).
- Pixelate exact home/work locations in history.
- Don't store driver location history beyond regulatory minimum.

### Common Mistakes

| Mistake | Fix |
|---------|-----|
| Lat-lng range query in Postgres at scale | Use H3 or geohash + Redis |
| Storing every location update | Stream to Kafka; archive to S3; keep only latest hot |
| Sync matching loop in API request | Async via queue; rider sees "matching..." state |
| One global matching DB | City-sharded; matching is local |
| Treating it like a CRUD app | The hard part is real-time geospatial + state machines |

> [!NOTE]
> The 80% of complexity is in **matching** and **state machines**. Payments are well-trodden territory; pricing is ML; the unique problem is "find a nearby driver and lock them in 5 seconds at planetary scale."

## Uber Deep Dive

### API Design

```
POST /v1/rides/estimate   { pickup, dropoff, product }  -> { fare_range, eta_pickup, surge, fare_token }
POST /v1/rides            headers: Idempotency-Key
                          { fare_token, pickup, dropoff, product, payment_profile_id }
                          -> 201 { ride_id, status: "MATCHING" }
GET  /v1/rides/{id}                         -> state, driver, vehicle, eta, polyline
POST /v1/rides/{id}/cancel
WS   /v1/stream (rider)    <- { ride_state, driver_loc, eta }
WS   /v1/stream (driver)   -> { loc, heading, speed, ts }   <- { offer, ride_update }
POST /v1/driver/offers/{offer_id}/accept | decline
POST /v1/driver/status { online | offline }
POST /v1/rides/{id}/start | end   (driver)
POST /v1/rides/{id}/sos | share   (safety)
```

`fare_token` is a signed, short-lived (about 2 min) upfront price from the estimate, so the price the rider saw is the price charged (upfront pricing), independent of route drift.

### Data Model

```sql
CREATE TABLE ride (
    ride_id        UUID PRIMARY KEY,
    rider_id       BIGINT NOT NULL,
    driver_id      BIGINT,
    city_id        INT NOT NULL,
    product        TEXT,                       -- X, XL, Pool, Black
    status         TEXT NOT NULL,              -- state machine
    version        INT NOT NULL DEFAULT 0,     -- optimistic concurrency
    pickup         POINT, dropoff POINT,
    fare_cents     INT, surge_multiplier NUMERIC(4,2),
    requested_at   TIMESTAMPTZ, matched_at TIMESTAMPTZ, started_at TIMESTAMPTZ, ended_at TIMESTAMPTZ,
    idem_key       TEXT, UNIQUE (rider_id, idem_key)
);
CREATE TABLE ride_event (ride_id UUID, seq INT, type TEXT, payload JSONB, at TIMESTAMPTZ,
                         PRIMARY KEY (ride_id, seq));        -- append-only audit log
CREATE TABLE driver_state (driver_id BIGINT PRIMARY KEY, status TEXT, cell BIGINT,
                           lat DOUBLE PRECISION, lng DOUBLE PRECISION, heading INT,
                           current_ride_id UUID, updated_at TIMESTAMPTZ);       -- hot KV in practice
CREATE TABLE offer (offer_id UUID PRIMARY KEY, ride_id UUID, driver_id BIGINT,
                    status TEXT, sent_at TIMESTAMPTZ, expires_at TIMESTAMPTZ);
```

At scale, the trip store is a sharded, city-partitioned datastore (Uber used Schemaless on MySQL, then Docstore); `ride_event` is the source of truth and `ride` a projection.

### ETA Service

ETA is called constantly: matching ranks by ETA (not straight-line distance), fares, and UI.

- **Road graph**: OSM or proprietary map data as a directed weighted graph: nodes = intersections, edges = road segments with speed, turn restrictions, one-ways.
- **Why not plain Dijkstra/A\***: continent-scale graph, tens of thousands of queries/s; naive search is too slow.
- **Contraction Hierarchies (CH)**: preprocessing orders nodes by importance and adds *shortcut* edges that skip less important nodes. Query = bidirectional Dijkstra over only "upward" edges, microseconds to a few ms. Cost: preprocessing and **weak support for changing weights** (traffic).
- **Customizable CH / Multi-Level Dijkstra (CRP)**: split the metric-independent topology (expensive preprocessing, done once) from the metric (edge weights, cheap re-customization in seconds). This is the production pattern for **live traffic**.
- **Live traffic**: weights = f(historical speed by segment x time-of-week, live GPS probe speeds from the driver pings, incidents). Map-matching snaps pings to segments (HMM-based), aggregates speeds per segment per minute (Kafka -> Flink), publishes a new metric version each N seconds.
- **Many-to-one ETA for matching**: rider to 50 candidate drivers: run one backward search from the rider or use a distance matrix (PHAST/one-to-many on CH), not 50 separate queries.
- **ML correction**: a model (gradient boosted trees, later deep nets) predicts residual error of the graph-routing ETA using time, weather, pickup-location features (airport pickup spots). Uber's DeepETA post-processes routing ETAs.
- **Partition by region**: the graph is sharded by city/region with an overlap; cache hot (cell-pair) ETAs for 30-60 s.
- **Fallbacks**: if the ETA service is slow, fall back to straight-line distance x speed factor for ranking only.

### Dispatch: Greedy vs Batched Global Matching

**Greedy (first-come, nearest driver)**: each request instantly takes the nearest available driver. Simple and low latency but locally optimal and globally poor: request A steals the driver that was the only good option for request B.

**Batched matching** (Uber and Lyft both do this): accumulate requests and idle drivers for a short window (about 1-3 s), then solve a **bipartite assignment** problem:

```
cost(r, d) = ETA(d -> pickup r) + penalties (low rating match, product mismatch, deadhead)
minimise  sum cost over a matching     (Hungarian algorithm O(n^3), or auction / min-cost-flow,
                                        sparsified by only allowing d within k-ring of r)
```

- Minimises total (or average) pickup time; improves marketplace efficiency, cuts cancellations.
- Window trade-off: longer window = better matches, worse request latency. Adaptive: longer in dense supply/demand, near zero when sparse.
- **Hungarian is cubic**: shard by city/H3 region, restrict candidates (top-10 per rider), solve many small problems in parallel; use approximations (greedy + local swaps, or auction) for huge cells.
- Include **future-value / driver positioning** in the cost: sending a driver far into a low-demand area has an opportunity cost (reinforcement-learning-derived value functions in research/production).
- Stability and fairness: avoid persistently starving certain drivers; ensure idle-time fairness.

### Driver Offer Fanout and Timeouts

- **Sequential offer**: send to best driver, wait about 10-15 s (driver app shows countdown); on decline/timeout, next. Predictable but slow tail.
- **Parallel (broadcast to K)**: offer to top K at once; first accept wins. Faster but wastes driver attention (they see offers vanish), more bookkeeping.
- **Hybrid**: top-1 or 2 first; widen if no response in 5 s.
- **Atomic accept**: the claim is a compare-and-set on the ride (`status=MATCHING -> MATCHED, driver=X`) and on the driver (`available -> reserved`). Use a ride-level version check or Redis Lua; loser gets "ride no longer available".
- Offer record with `expires_at`; a timer (or scheduled event) marks expired and triggers re-dispatch. Late accept after expiry is rejected.
- **Reserve the driver** while an offer is outstanding so batched matching doesn't offer him to two riders (a driver holds at most one offer at a time).
- **Acceptance-rate modelling**: rank by P(accept) x quality; drivers who often decline get lower priority. Show destination/fare to inform decisions, subject to fairness rules.
- Push delivery via persistent connection plus APNs/FCM fallback with high priority; track delivered/seen.

### Surge Pricing Math

Goal: balance supply and demand per area/time: raise price so some demand drops and more drivers move in.

```
Per H3 cell (res 7-8), per 1-2 min window:
  demand  = unfulfilled requests + app opens (smoothed EWMA)
  supply  = idle drivers + drivers about to finish trips (ETA < 2 min)
  ratio   = demand / max(supply, eps)
  multiplier = clamp( f(ratio), 1.0, cap )        # f monotone, e.g. 1 + a*(ratio - 1)^b
```

- **Smoothing**: temporal EWMA and spatial smoothing across neighbouring cells (avoid price cliffs at cell borders, driver gaming); hysteresis so multiplier doesn't flap.
- **Elasticity-based**: modern systems estimate demand elasticity and solve for the multiplier that clears the market: choose m so `expected_requests(m) ~= expected_supply(m)`; ML forecasts of next 10-15 min demand.
- **Caps and regulations**: emergencies (price-gouging laws), city caps, rounding to 0.1x.
- **Upfront pricing**: price computed at estimate time = base + distance/time model x surge, locked via `fare_token`. The platform eats variance (route longer, traffic).
- **Driver side**: heatmaps, earnings guarantees, boosts in place of visible multipliers in some markets.
- Pipeline: Kafka (requests, driver-states) -> Flink windowed aggregation -> multiplier store (Redis, cell -> multiplier) read by pricing service.

### Pooling / Shared Rides

Match multiple riders on overlapping routes into one vehicle.

- **Candidate generation**: for a new request, find in-flight pool trips (and idle drivers) where detour is small.
- **Insertion heuristic**: for a vehicle with current route [stops], try inserting the new rider's pickup and dropoff at every position pair (i<j): cost = added time for existing riders and the new rider; constraints: each rider's max detour (e.g. 1.4x direct time or +8 min), vehicle capacity, pickup wait limit. Choose min cost feasible insertion, O(stops^2) per candidate.
- **Batched** (like dispatch): solve assignment of requests to vehicles with route insertion cost using a time window (a few seconds); VRP-with-time-windows flavour; heuristics (insertion, local search), not exact.
- **Pricing**: discounted for all, share price depends on a 'shared distance' model; guarantee that the price never exceeds the solo fare.
- **Fixed pickup points / walking** (Express Pool) reduce detours and make matches more likely.
- **Trade-offs**: higher utilisation vs rider wait and detour variance; poor at low density (use solo fallback within N seconds).
- ETA service must support fast many-to-many (insertion evaluation does many path-time queries); precompute matrices between candidate stops.

### Sharding, Ringpop, and Hot Cells

- **City as the unit**: matching is local, so each city (or region cluster) is an independent shard; request routed by pickup location to the owning matching workers.
- **Ringpop** (Uber): a consistent-hash ring with gossip membership (SWIM) embedded in app servers; requests for a key (city/cell/driver) are forwarded to the owning node, which keeps the state in memory (no remote cache hop). Node failure: ring rebalances, ownership moves, state rebuilt from the Kafka log and the next pings (every 4 s). Alternatives today: sticky routing with a Kafka-partition-per-shard model or actor frameworks.
- **Ownership key**: by H3 cell (res ~5-6) so geo queries mostly hit one node; cell-boundary queries touch neighbours (scatter to few shards).
- **Hot cell** (stadium, airport, New Year's Eve): single owner overloaded. Mitigate: split into finer sub-cells across nodes, replicate read state, shard by `driver_id hash` inside the cell for writes, queue and batch requests, airport **virtual queues** (FIFO driver lots with geofence).
- State per node is soft: location is recreated within seconds, which is why losing a node is tolerable (no durable state needed for positions).

### Location Ingest at 2.5M updates/s

```
Drivers ── TLS ─► regional LB ─► WS/gRPC gateways (stateless, connection-holding)
                                   │  validate, rate limit, batch
                                   ▼
                              Kafka (partition key = driver_id/city), compact + raw topics
                    ┌──────────────┼───────────────────┬────────────────┐
                    ▼              ▼                   ▼                ▼
          Geo-index updater   ETA/traffic probe   Trip tracker     Archive (S3/Parquet)
          (in-memory H3 map)   (Flink map-match)  (polyline, fares)  ML, analytics
```

- Throughput: 2.5M msgs x ~100 B = 250 MB/s: trivial for Kafka across hundreds of partitions; the cost is connection count (millions of persistent sockets, so tens of thousands of gateway instances worldwide, epoll/Netty, protobuf).
- **Adaptive frequency**: 4 s idle, 1 s during trips, less when stationary (battery/data); client batches (several samples per message), sends deltas.
- **Write only latest to hot store**; history asynchronous. Skip updates that don't change the cell for index maintenance.
- Backpressure: shed lowest-value (idle-driver) pings under overload, never trip pings.
- Dedup and order using `(driver_id, seq/ts)`; reject out-of-order or implausible jumps (GPS spoofing check using speed limit).
- Fine-grained **trip location log** stored for billing disputes and safety (retention policy).

### Trip-State Consistency

Trip state is money; the rider and driver apps, billing and dispatch must agree.

- **Single writer per ride**: all transitions for a ride route to its owner shard (by `ride_id`); each transition is a conditional write `WHERE status = expected AND version = n`.
- **Event log first**: append `ride_event` (seq) in the same transaction as the projection update; downstream consumers (billing, notifications, ETA, analytics) read events from Kafka via the **outbox**.
- **Idempotent commands** with idempotency keys (accept, start, end) because mobile networks retry; `end` called twice must not double charge.
- **Conflict resolution** for offline devices: the driver app may start/end trips while offline; reconcile by event timestamps and server-side validation (GPS trace). Prefer server-authoritative state; device-originated events are claims, validated.
- **Cross-service consistency**: saga, not distributed transaction: `ride ended -> fare computed -> payment capture -> receipt`; each step idempotent, with compensations (refund, adjust).
- **Reconciliation**: detect rides stuck in `in_trip` too long, rider and driver disagreeing, zero-distance trips.

### Payments Split

- At trip end: compute fare = base + distance + time + surge + tolls + fees, minus promos; rider charged via PSP (authorization at request or start; capture at end; retry/fallback to other instruments; handle declines with debt/collection flows).
- **Marketplace split**: platform commission (e.g. 25%), driver earnings, taxes, tips, tolls pass-through, safe-rides fee, booking fee. Recorded as **double-entry ledger** entries (rider payable, driver earnings, platform revenue, tax liabilities).
- **Payouts**: instant cash-out (debit rails, fee) or weekly batch (ACH/UPI/NEFT). Balances in a ledger service; payouts idempotent per batch id.
- **Split fare** (rider-to-rider): fare divided among invited riders, each charged separately; failure of one share handled by the initiator.
- **Pool**: each rider's share priced individually; driver gets sum minus commission.
- **Multi-currency / tax** per jurisdiction; invoices and driver statements; refunds and adjustments as new ledger entries (never mutate).
- Cash trips (India etc.): driver collects cash; platform commission accumulates as driver debt netted against later digital earnings.

### Fraud and Abuse

| Fraud | Defence |
|-------|---------|
| GPS spoofing drivers (fake long trips, surge chasing, phantom drivers) | Sensor consistency (accelerometer, cell/WiFi vs GPS), impossible-speed checks, mock-location detection, trip path vs map |
| Rider-driver collusion (promos, fake trips) | Graph analysis, repeated pair detection, device/payment linkage, delayed payouts |
| Stolen cards | PSP risk, 3DS, first-trip limits, velocity, device fingerprint |
| Account takeover | MFA, device binding, anomaly detection |
| Fake accounts for promo abuse | Phone/ID verification, device clustering, promo limits |
| Driver cancelling to cherry-pick | Acceptance/cancellation scoring, penalties |
| Cash-out fraud | Cash-out velocity limits, holds on new accounts |

Real-time scoring (feature store, rules plus ML) in the request path for risky actions; offline graph models; manual review queues.

### Safety Features

- **Share trip** (live link), **SOS / emergency button** (call local emergency number with location, in-app incident team), PIN verification at pickup (rider tells a 4-digit PIN), **RideCheck** (anomaly detection: long unexpected stop, route deviation, possible crash from accelerometer) triggering check-in prompts.
- **Driver ID verification** (selfie liveness at login, background checks), vehicle checks.
- **Number masking** (proxy calls/SMS), pickup-location obfuscation after trip, **audio recording** options (encrypted, stored on device, released only on report).
- Trusted contacts, women-preferred matching options (regulatory), night-ride rules.
- Incident pipeline: priority queue to safety ops, SLAs, evidence retention.

### Scheduled Rides

- Reservation up to N days ahead: store `scheduled_for`, pickup, product; guarantee is soft (marketplace) or hard (premium products).
- **Dispatch timing**: a scheduler fires at `T - lead` (lead depends on predicted supply and ETA, e.g. 10-30 min before) to start matching; pre-assign driver earlier for airport/early-morning (driver commits, gets guaranteed minimum or bonus).
- Data: delayed-job store (time-bucketed partitioned table plus a scheduler workers scanning due buckets, or delayed queue); idempotent firing keyed by ride id.
- Handle failure: if no driver by T-minus X, escalate offers with higher incentives, notify the rider with options, cancel with no fee/credit.
- Capacity planning: forecast of scheduled demand informs driver incentives.

### Observability and Testing

- SLIs: request-to-match latency, match rate, ETA accuracy (predicted vs actual, MAPE), dispatch cost (avg pickup ETA), cancellation rate, offer acceptance rate, surge stability, payment success rate, state-stuck counters.
- **Simulation** is core: replay real historical request/driver logs into a **marketplace simulator** to test matching, pricing and pooling algorithm changes offline before experiments.
- Experimentation: **switchback tests** (alternate algorithm by city/time-block) since riders and drivers interact (interference breaks user-level A/B).
- Chaos: region/city failovers, Kafka lag, ring membership churn, GPS noise injection; load tests of event spikes (bars closing, concert end).

### Interview Follow-ups

- *"How do you keep driver location private to the matched rider only?"* — Server enforces ACL; only stream to a rider's WS during the trip duration.
- *"How do you handle a driver who can't be reached after acceptance?"* — Timeout on `matched` state (30s); reassign; possibly penalize driver.
- *"What's H3 vs S2?"* — Both are hierarchical geospatial indexes. H3 = hexagons (Uber). S2 = squares (Google). Hexagons have uniform distance to all 6 neighbors; squares have non-uniform diagonals.
- *"How is ETA computed at scale?"* — Road graph with Contraction Hierarchies or Customizable CH (separate topology from live traffic weights), live probe speeds from driver pings via map-matching and Flink, ML correction on top; one-to-many queries for candidate drivers.
- *"Why not just run Dijkstra or A\* per request?"* — Too slow on continental graphs at tens of thousands of queries per second; CH gives sub-millisecond queries after preprocessing; plain CH cannot cheaply absorb traffic changes, so use CCH/CRP.
- *"How do you incorporate live traffic into contraction hierarchies?"* — Customizable CH: expensive node ordering done once on topology; cheap re-customization of shortcut weights every few seconds with the new traffic metric version.
- *"Greedy nearest-driver vs batched matching?"* — Greedy is fast but globally suboptimal; batching in 1-3 s windows and solving bipartite assignment (Hungarian / min-cost flow / auction) minimizes total pickup time and cancellations at the cost of slight added latency.
- *"Hungarian is O(n^3); how does it scale?"* — Shard by city/H3 region, sparsify (top-k candidates per rider), solve small independent problems in parallel, approximate with auction or greedy-plus-swaps in giant cells; adaptive window size.
- *"How does a driver offer work with timeouts?"* — Offer record with expiry (10-15 s); reserve the driver so he gets one offer at a time; accept is atomic CAS on the ride; on timeout/decline, re-dispatch; late accepts rejected.
- *"Sequential or parallel offers?"* — Sequential is cleaner for drivers but adds latency; parallel to top-K is faster but wastes attention; hybrid widen-after-5s is typical; rank by P(accept) x ETA.
- *"What if two drivers accept simultaneously?"* — Conditional update on ride (`status=MATCHING`) is linearizable per ride on its owning shard; one wins, the other gets 'ride taken' and is released.
- *"How does surge pricing work?"* — Per H3 cell and window compute demand/supply imbalance, smooth over time and space, map through a monotone function (or elasticity-based clearing price) with caps; store cell multiplier in Redis; lock price into upfront fare token.
- *"How do you avoid surge flapping and border gaming?"* — EWMA smoothing, hysteresis, spatial smoothing among neighbouring cells, rate limits on multiplier change, fare locked at quote.
- *"Is surge fair/legal?"* — Caps during emergencies (price-gouging laws), transparency, upfront price; regulators in some cities set maximum multipliers.
- *"How does UberPool/shared ride matching work?"* — Insertion heuristic into existing routes with detour and wait constraints, batched assignment over a short window, fixed pickup points, discount guarantee; fall back to solo if no match within N seconds.
- *"What's the complexity of pooling and how do you tame it?"* — Insertion is O(stops^2) per vehicle-request pair; prune by geo/time proximity first, cap vehicle stops, use cached travel-time matrices, local search heuristics rather than exact VRP.
- *"What is Ringpop and why does Uber use it?"* — Application-layer consistent hashing with gossip (SWIM) membership so each node owns a key range (city/cell) and keeps its state in memory; requests forwarded to owner; no external cache hop; state soft and rebuildable from pings and Kafka.
- *"What happens when a shard owner dies?"* — Gossip detects failure, ring rebalances, new owner rebuilds state from the Kafka log and the next pings within seconds; in-flight trips are durable in the trip store so rides are not lost.
- *"How do you handle a hot cell like an airport or stadium?"* — Split cell into finer H3 sub-cells across nodes, shard writes by driver hash, replicate read state, queue/batch requests, airport geofenced FIFO driver queues, pre-position drivers with incentives.
- *"How do you ingest 2.5M location updates per second?"* — Persistent WebSocket/gRPC gateways, batched protobuf, adaptive frequency, Kafka partitioned by driver/city, in-memory geo index updated by consumers, async history to S3; shed idle-driver pings under overload.
- *"How do you stop GPS spoofing?"* — Mock-location detection, sensor fusion, impossible speed/jump checks, map-matching plausibility, device attestation, cross-checks with cell/WiFi, penalties and bans.
- *"How do you ensure trip state is consistent between rider app, driver app and billing?"* — Single writer per ride shard, conditional state transitions, event-sourced `ride_event` with outbox to Kafka, idempotent commands, server-authoritative state with client claims validated.
- *"What if the driver's phone is offline when the trip ends?"* — Device queues events with timestamps and sync on reconnect; server reconstructs trip from GPS trace; fare from upfront price; cap re-computation to prevent abuse.
- *"How do you prevent double-charging a rider?"* — Idempotency key per ride-payment, ledger entries unique per `(ride_id, type)`, PSP idempotent calls, reconciliation against PSP statements.
- *"How do you design the payments split?"* — Double-entry ledger: rider charge, platform commission, driver earnings, taxes, tips, tolls; payouts batched or instant; refunds as compensating entries; cash trips net commission against future digital earnings.
- *"Upfront pricing: who bears the risk?"* — Platform: price from estimate (route/time/surge model) is locked; deviations absorbed unless route materially changes (rider changes destination -> reprice); calibrated models keep average risk near zero.
- *"How do scheduled rides work?"* — Persist the booking; scheduler fires at `T - lead` to begin matching (lead predicted from supply/ETA), optionally pre-assign a committed driver with a guarantee; escalate incentives and notify on failure.
- *"How do you implement the delayed scheduler reliably?"* — Time-bucketed table with workers scanning due rows using `SKIP LOCKED` or a delayed queue; fire idempotently keyed by ride id; handle clock via DB time; redundant scanning for HA.
- *"How do you detect fraud like rider-driver collusion?"* — Graph analytics on repeat pairs and shared devices/cards, anomalous short high-promo trips, trip path validation, delayed payouts and holds, ML scoring in the loop for risky actions.
- *"What safety features are expected?"* — Live trip sharing, SOS with location, PIN verification, RideCheck anomaly detection (route deviation, long stop, crash), masked calls, driver ID liveness, background checks, audio recording options, incident ops pipeline.
- *"How do you do cancellation fees fairly?"* — Fee after grace period (e.g. 2 min) or when driver is near/arrived; waive if driver at fault (wrong location, long delay); driver compensated; abuse scoring for riders who cancel repeatedly.
- *"How do you pick matching objective (ETA vs revenue vs fairness)?"* — Cost function combining pickup ETA, cancellation probability, driver idle-time fairness, and future-value positioning; tune with simulation and switchback experiments.
- *"How do you test marketplace changes?"* — Offline simulator replaying historical data, then **switchback** or geo-split experiments because user-level A/B suffers interference (shared supply).
- *"How do you handle multi-region and data residency?"* — City-pinned shards in the nearest region; regional data stores for PII; cross-region only for global identity/auth; regional failover pairs; GDPR/DPDP retention for location history.
- *"How do you handle driver incentives and guarantees at scale?"* — Event stream aggregates trips per driver; rule engine computes quest/bonus progress; ledger entries; anti-fraud checks before payout.
- *"What do you cache and where?"* — Geo index and driver state in-memory on shard owners; surge multipliers in Redis; ETA results by cell pair for 30-60 s; map tiles and static config on CDN.
- *"How would you degrade gracefully during an outage?"* — Disable pool and scheduled rides first, switch to greedy dispatch, fall back to straight-line ETA, serve cached surge, queue trip-end billing for later; protect core request->match->trip path.
- *"How do you compute accurate fares if GPS is noisy?"* — Map-match the trace, snap to road network, use odometer/telematics when available; filter outliers; compare with route estimate; for upfront price mismatch, fare stays locked unless the rider changed the route.
- *"How do you support multiple products (X, XL, Black, Auto, Bike)?"* — Product as a filter/constraint on supply (vehicle capacity, class, licence), separate pricing models and commission rules, shared dispatch with product-aware cost; supply can be eligible for multiple products with priority rules.
- *"How do you handle drivers working multiple apps (multi-homing)?"* — Faster offer timeouts, acceptance modelling, incentives, reservation of driver across offers, handle 'driver already on another platform trip' via cancellations and cooldown.
