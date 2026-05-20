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

### Interview Follow-ups

- *"How do you keep driver location private to the matched rider only?"* — Server enforces ACL; only stream to a rider's WS during the trip duration.
- *"How do you handle a driver who can't be reached after acceptance?"* — Timeout on `matched` state (30s); reassign; possibly penalize driver.
- *"What's H3 vs S2?"* — Both are hierarchical geospatial indexes. H3 = hexagons (Uber). S2 = squares (Google). Hexagons have uniform distance to all 6 neighbors; squares have non-uniform diagonals.
