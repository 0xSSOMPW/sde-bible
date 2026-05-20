# Q: Database indexing — B-tree, hash, GIN, covering, composite. Trade-offs.

**Answer:**

Indexes turn `O(N)` table scans into `O(log N)` lookups. Each one costs disk space + write amplification. Understanding which index helps which query is the difference between a 5 ms response and a 5 second one.

### B-Tree (default)

```
                [50, 80]
               /    |   \
        [20,30,40] [60,70] [85,90,99]
```

Sorted, balanced tree. Logarithmic depth. Used by:
- Postgres (default)
- MySQL InnoDB primary + secondary
- Most RDBMS

**Supports**:
- Equality: `=`
- Range: `<`, `>`, `BETWEEN`
- Prefix wildcard: `LIKE 'abc%'` (not `'%abc'`)
- `ORDER BY` matching index direction

**Doesn't help**:
- Functions: `WHERE lower(email) = ?` (use functional index)
- Suffix wildcards: `LIKE '%abc'` (use trigram or reverse index)
- `<>` (not equal) — index scanned but rarely useful

### Hash Index

```
hash(key) → bucket → entry
```

**Pros**: O(1) lookup.
**Cons**:
- No range queries.
- No ordering.

Postgres has hash indexes but rarely worth it over B-tree. Used mostly internally (e.g., in-memory hash joins).

### Composite Index

Index on multiple columns, ordered:

```sql
CREATE INDEX ON orders (user_id, created_at DESC);
```

Helps queries that filter by the **leftmost prefix**:

```sql
WHERE user_id = ?                              -- ✅ uses index
WHERE user_id = ? AND created_at > ?           -- ✅
WHERE user_id = ? ORDER BY created_at DESC     -- ✅ no sort needed
WHERE created_at > ?                           -- ❌ doesn't help
```

Order matters. Lead with the most selective column you filter on.

### Covering Index (Index-Only Scan)

If the index includes all columns a query needs, the DB never reads the table:

```sql
CREATE INDEX ON orders (user_id, created_at) INCLUDE (status, total);

SELECT user_id, created_at, status, total
FROM orders
WHERE user_id = ?;
-- index-only scan; skip table
```

Postgres `INCLUDE` and MySQL InnoDB cluster keys give this.

### Partial Index

Index a subset of rows:

```sql
CREATE INDEX ON orders (created_at) WHERE status = 'pending';
```

Smaller, faster for queries that match the `WHERE`. Useful for hot subsets (pending jobs, active users).

### Functional / Expression Index

```sql
CREATE INDEX ON users (lower(email));

SELECT * FROM users WHERE lower(email) = ?;     -- uses index
```

Needed when the query applies a function to a column. Without it, the function runs per row.

### GIN (Generalized Inverted Index)

For multi-valued columns: arrays, JSONB, text-search.

```sql
CREATE INDEX ON orders USING gin (tags);
SELECT * FROM orders WHERE tags @> ARRAY['urgent'];

CREATE INDEX ON docs USING gin (to_tsvector('english', body));
SELECT * FROM docs WHERE to_tsvector('english', body) @@ to_tsquery('postgres');
```

Each tag/word becomes an inverted-index entry pointing to rows containing it.

### GIST (Generalized Search Tree)

Tree-based, supports complex types:
- Geometry (PostGIS).
- Range types.
- Full-text search (older).

```sql
CREATE INDEX ON places USING gist (location);
SELECT * FROM places WHERE ST_DWithin(location, ?, 1000);
```

### BRIN (Block Range INdex)

Tiny index storing per-block summary stats (min/max). Useful for big tables where rows are naturally ordered (timestamps):

```sql
CREATE INDEX ON events USING brin (created_at);
```

Small index, helps range scans on time-ordered tables. Bad for random-order data.

### Cost of an Index

Every index adds:
- Disk space (~10–30% of table).
- Write amplification: inserts and updates must update all relevant indexes.
- Memory: cached pages in the buffer pool.

Indexes are not free. Don't index columns that aren't filtered/sorted.

### When NOT to Index

- Low-cardinality columns (`gender`, `status` with 3 values). Postgres planner often prefers seq scan.
- Frequently updated columns (especially in hot tables).
- Tables < ~10k rows (seq scan is faster anyway).

### Reading Query Plans

Postgres:

```sql
EXPLAIN ANALYZE SELECT * FROM orders WHERE user_id = 42;
```

Look for:
- `Index Scan` (good).
- `Index Only Scan` (best).
- `Seq Scan` on a big table (smell).
- `Bitmap Heap Scan` (often fine for multi-row results).
- `Sort` step (consider an index that returns pre-sorted).

### Index Maintenance

- **REINDEX**: rebuild bloated indexes (Postgres has `REINDEX CONCURRENTLY` since 12).
- **Statistics**: `ANALYZE` updates table statistics; planner uses them.
- **Auto-vacuum**: tune for write-heavy tables.

### Sharded Indexes

In sharded systems:
- **Local index**: each shard's own index. Fast for queries that include shard key.
- **Global secondary index**: separate store keyed by non-shard column; cross-shard queries → fan out vs centralized index.

DynamoDB GSI is the latter; Cassandra secondary index is the former.

### Common Mistakes

| Mistake | Reality |
|---------|---------|
| One index per column | Many redundant; combine into composite |
| Indexing every column "to be safe" | Write amplification; storage waste |
| Wrong column order in composite | Index unused; lead with most-filtered |
| Missing index on FK columns | Joins do seq scan |
| Index on UUID v4 primary key (random) | Random inserts → page splits, bloat. Use sortable UUIDs (ULID, UUIDv7) |

> [!NOTE]
> The most useful skill is reading `EXPLAIN ANALYZE`. Indexes are not magic; the planner picks one only if it thinks it'll help. Verify, don't assume.

### Interview Follow-ups

- *"How would you index `WHERE status = 'active' AND created_at > ?`?"* — Partial index on `(created_at)` `WHERE status = 'active'`; or composite `(status, created_at)`.
- *"How does MySQL InnoDB primary key clustering differ from Postgres?"* — InnoDB stores rows physically in PK order — primary key lookup is the table. Postgres stores rows in heap; index is separate.
- *"When would you choose hash over B-tree?"* — Equality-only queries with very high cardinality and no range needs; in practice, rarely.
