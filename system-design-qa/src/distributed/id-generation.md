# Q: Unique ID generation — UUID, Snowflake, ULID, KSUID. Pick one.

**Answer:**

Every distributed system needs IDs. The naive `auto_increment` doesn't work across nodes; UUID v4 is random but unsorted. Modern choices like **Snowflake**, **ULID**, **UUIDv7** combine **uniqueness**, **sortability**, and **decentralization**.

### Requirements for a Good ID

- **Unique**: no collisions across distributed generators.
- **K-sortable**: rough time-ordering helps DB indexes and pagination.
- **Compact**: 16 bytes is fine; 64-bit is great.
- **Stateless generation**: no coordination per ID.
- **Opaque externally**: don't leak business semantics.

### Options

| Scheme | Bits | Sortable | Coordination | Notes |
|--------|------|---------|--------------|-------|
| `auto_increment` | 32–64 | yes | DB row lock | Centralized |
| **UUID v1** | 128 | partial (MAC-time) | none | Leaks MAC address |
| **UUID v4** | 128 | NO (random) | none | Universal default but unsortable |
| **UUID v7** | 128 | yes (time-ordered) | none | New; modern default |
| **Snowflake** | 64 | yes | machine-id config | Twitter's scheme |
| **ULID** | 128 | yes | none | URL-safe encoding |
| **KSUID** | 160 | yes | none | Segment's scheme |
| **NanoID** | configurable | no | none | URL-safe random; like UUIDv4 |

### UUID v4

```
hex: f47ac10b-58cc-4372-a567-0e02b2c3d479
```

128 bits, 122 of randomness. Collision probability negligible.

Cons:
- Not time-ordered → bad for DB index locality (random inserts → page splits in B-tree).
- Larger than 64-bit IDs.
- Hard for humans to glance.

### UUID v7

```
018f8d34-... (first 48 bits = ms timestamp; rest random)
```

Released as draft-2024, in mainstream libs (Postgres `uuid_generate_v7()` via extensions).

Pros:
- 128 bits like UUID.
- Time-ordered: index locality, easy "newest first" sort.
- Drop-in replacement for UUID v4.

**The modern default** for new systems. Combines UUID's universality with sortability.

### Snowflake

64-bit ID:

```
| 1 sign | 41 timestamp (ms) | 10 machine_id | 12 sequence |
```

- Timestamp: ms since epoch (Twitter's: 1288834974657).
- Machine ID: configured per node (10 bits = 1024 machines).
- Sequence: counter per ms per node.

Properties:
- 64-bit, compact.
- K-sortable (by timestamp).
- 4096 IDs/ms/machine = 4M IDs/sec/machine.
- Requires unique machine IDs.

Used by Twitter, Discord, Instagram (similar scheme).

Bookkeeping:
- Machine IDs from etcd / ZK / DB.
- Clock skew handling: refuse to generate if clock goes backwards.

### ULID

128-bit, lexicographically sortable:

```
01H6FXM34VWCEK0NHJRA94QSEW
^^^^^^^^^^                  Crockford-base32 timestamp (48 bits)
          ^^^^^^^^^^^^^^^^   Random (80 bits)
```

Pros:
- Time-ordered.
- URL-safe.
- No coordination.
- Same size as UUID.

Library: lots; e.g., `ulid` in many languages.

### KSUID (Segment)

20 bytes (160 bits):

```
27-char base62: 0o5sQohJzs8WW3hjwOyaCfaTeZB
```

Like ULID but bigger (more random bits → effectively no collision risk at any scale).

### Sortability and DB Performance

B-tree primary key with random IDs (UUID v4):
- Each insert → random page in the index.
- Pages split, fragment, bloat.
- Cache locality poor.
- Insert-heavy workloads slow.

Sortable IDs (UUIDv7, ULID, Snowflake, auto_increment):
- Each insert → end of index (last page).
- Sequential writes; cache-friendly.
- 2–10× faster insert throughput in benchmarks.

For new systems: use sortable IDs.

### Cross-Region / Multi-DC

- Snowflake: assign machine IDs per region (e.g., first 3 bits = region).
- UUIDv7 / ULID: no coordination at all.
- DB-managed auto_increment: doesn't work across regions.

### Don't Expose Internal IDs

Sequential / sortable IDs leak:
- Total count (`/orders/12345` → estimate Twitter's order volume).
- Creation order (privacy / business).

For external APIs: use opaque IDs (UUID or signed hashids). Internal: keep the sortable ones.

### Choosing in 30 Seconds

```
External API URL identifier   → UUIDv4 or signed hashids
Internal primary key, new   → UUIDv7 or ULID
Already on Snowflake      → keep
Distributed numbering at extreme scale (10M+ IDs/sec)  → Snowflake-class
URL-friendly string         → ULID or NanoID
```

### Common Mistakes

| Mistake | Reality |
|---------|---------|
| UUID v4 PK on a big table | Index bloat, slow inserts |
| Snowflake with same machine ID on multiple nodes | Duplicate IDs |
| Exposing auto_increment in URLs | Leaks order volume |
| Generating IDs in client code | Trust issues; clients can forge |
| Mixing schemes within one table | Painful migrations |

> [!NOTE]
> For 2025 greenfield work, default to **UUIDv7** unless you specifically need 64-bit (Snowflake) or URL ergonomics (ULID). UUIDv7 is what UUID v4 should have been.

### Interview Follow-ups

- *"How does Twitter generate IDs?"* — Snowflake; each datacenter has a service that emits IDs with its assigned machine_id range.
- *"What if a node's clock jumps backward?"* — Refuse to generate (Twitter's choice); panic vs reject; tolerate small skew via NTP.
- *"How do you migrate from int64 to UUID without downtime?"* — Add UUID column nullable; backfill; flip writes to populate both; switch reads; drop old.
