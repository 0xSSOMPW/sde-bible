# Q: Design a geo-proximity service (Yelp / "find restaurants near me").

**Answer:**

Spatial-index problem. Tests **geo-hashing / quad-trees / R-trees**, **trade-offs between range query and point-in-bbox**, and **scale considerations**.

### Requirements

**Functional**:
- Find places within radius R of (lat, lng).
- Filter by category, hours, rating.
- Sort by distance + ranking.
- Return a few hundred results.

**Non-functional**:
- 200M users, 10M places.
- p99 < 200 ms.
- High read, write rare (places change slowly).

### Spatial Index Options

| Structure | Best for | Notes |
|-----------|----------|-------|
| **Geohash** | KV / Redis | Hierarchical string keys |
| **H3 / S2** | Uniform cells | Hexagons (H3) / squares (S2); used by Uber, Google |
| **Quad-tree** | Adaptive density | Cells split when too many points |
| **R-tree** | Bounding boxes | Used in Postgres GIST, Elasticsearch |
| **KD-tree** | Small, in-memory | Bad for high dimensions |
| **PostGIS** | Postgres extension | Production-grade SQL spatial |

### Geohash Primer

Geohash encodes (lat, lng) → string. Prefix length controls cell size:

```
length 1: ~5,000 km cell
length 4: ~20 km
length 5: ~5 km
length 6: ~1 km
length 7: ~150 m
length 8: ~38 m
```

Same prefix = nearby. Inverse: query by prefix returns nearby points.

```
place "dr5ru..." (NYC)
query "find places near dr5ru..."
  → check cell "dr5ru" + 8 neighbors
  → filter by exact distance
```

### Architecture

```
client ──► API ──► spatial index ──► metadata DB
                       │                    │
                       ▼                    ▼
                Redis (geo)        Postgres (places)
                or Elasticsearch    or DynamoDB
```

### Storage

**Place metadata** (Postgres):
```sql
places (
  id, name, lat, lng, category, hours, rating, ...
)
```

**Spatial index** (Redis, Elasticsearch, or PostGIS):

Redis GEO commands (uses geohash internally):
```
GEOADD places 40.7128 -74.0060 "place_42"
GEOSEARCH places FROMLONLAT -74.0060 40.7128 BYRADIUS 1 km ASC COUNT 100
```

Elasticsearch `geo_point` field:
```json
{ "query": { "geo_distance": { "distance": "1km", "location": {...} } } }
```

PostGIS:
```sql
SELECT * FROM places WHERE ST_DWithin(
  geom, ST_MakePoint(-74.0060, 40.7128)::geography, 1000
);
```

For very high QPS, Redis GEO or Elasticsearch wins.

### Quad-Tree

Recursively split cells when too many points:

```
+----------+----------+
|    A     |    B     |        cell B is dense — split:
|          |          |        +----+----+
+----------+----------+        | B1 | B2 |
|    C     |    D     |        +----+----+
|          |          |        | B3 | B4 |
+----------+----------+        +----+----+
```

Pros: adapts to data density.
Cons: tree balancing on updates.

Used historically; modern systems often prefer fixed-cell H3 / S2 + per-cell sorted set.

### Query Algorithm

```
def find_nearby(lat, lng, radius_m, filters):
    # 1. determine cell precision for radius
    precision = pick_geohash_precision(radius_m)

    # 2. compute target cell + neighbors
    cells = neighbors_within_radius(geohash(lat, lng, precision), radius_m)

    # 3. fetch candidates from each cell
    candidates = []
    for cell in cells:
        candidates.extend(redis.smembers(f"cell:{cell}"))

    # 4. fetch metadata, filter, sort by distance
    places = db.batch_get(candidates)
    results = [(p, distance(p, lat, lng)) for p in places
                if matches_filters(p, filters) and distance(p, lat, lng) <= radius_m]
    results.sort(key=lambda x: x[1])
    return results[:limit]
```

Hot path is the cell membership lookup. With Redis: microseconds.

### Filtering

Pre-filter at the index when possible:
- Category-specific index: `cell:dr5ru:restaurant`.
- Open-now filter applied in app (depends on time of query).

Many filters → split indexes by major category. Trade index storage for query speed.

### Density Skew

Manhattan has 100× more places per cell than rural areas. Pure fixed-grid → some cells huge.

Mitigations:
- Adaptive grid (quad-tree).
- Multi-level indexing (broad → narrow).
- Limit candidates per cell, defer to follow-up query.

### Write Path

```
POST /place
  ├── INSERT INTO places (Postgres)
  ├── GEOADD to Redis
  └── async index into Elasticsearch for text search
```

Writes are rare (10M places, low add rate). Don't optimize.

### Read Path

```
GET /search?lat=...&lng=...&radius=...&category=...
  ├── lookup index (Redis GEO)
  ├── filter + rerank
  └── return JSON with distance + place data
```

### Pagination

Cursor-based: `cursor = last_distance + last_id`. Stable across writes.

### Caching

Identical queries (same lat-lng-radius bucket) hit cache:

```
cache_key = hash(geohash(lat,lng,7), radius, filters)
TTL ~30s for popular spots
```

CDN cache for non-personalized queries; sub-second response on hits.

### Ranking Beyond Distance

Yelp ranks by:
- Distance.
- Rating.
- Review count.
- Sponsorship.
- Personalization.

Cap candidates at e.g. 200 by distance; ML rerank top set.

### Scaling

- Index in Redis Cluster — shard by geohash prefix.
- Hot cities (NYC, LA) get more shards.
- Cross-region: each region has full index; data replicated async.

### Failure Modes

| Failure | Handling |
|---------|---------|
| Redis shard down | Fallback to Postgres+PostGIS (slower); alert |
| One cell hot (Times Square) | Sub-shard by sub-precision |
| Place data drifted between Postgres and index | Reconcile via daily job |

### Common Mistakes

| Mistake | Fix |
|---------|-----|
| Lat-lng full-scan filter on every request | Use a spatial index |
| Including all metadata in spatial cell | Cell stores IDs only; metadata separate |
| Single global cell size | Adaptive or hierarchical |
| Computing Haversine for every place ever | Pre-filter by cell first |

> [!NOTE]
> The defining decision is the **cell granularity** vs cell-membership cost trade. Too small = many cells per query; too big = filtering huge candidate lists. Tune per traffic shape.

### Interview Follow-ups

- *"How would you serve millions of moving entities (Uber drivers)?"* — Same idea, but live updates: each entity updates its cell on movement; query a cell + neighbors. Tile38 / Redis GEO scale to this.
- *"How does Yelp differ from a maps service?"* — Maps stores roads, polygons, terrain (much richer). Yelp is point-of-interest with attributes.
- *"What about indoor / floor-level proximity?"* — Add altitude/floor to the index; usually a separate dataset because indoor maps are richer than (lat, lng).
