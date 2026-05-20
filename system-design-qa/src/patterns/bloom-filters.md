# Q: Bloom filters and probabilistic data structures — when to use them.

**Answer:**

Probabilistic data structures answer set-membership and cardinality questions with **much less memory** than exact structures, at the cost of small, well-defined error rates. Bloom filters are the most famous; their cousins (HyperLogLog, Count-Min Sketch, Cuckoo filters) handle related problems.

### Bloom Filter

A bit array + K hash functions. Answers "is X possibly in the set?":

```
add(x):
    for i in 1..K:
        bits[hash_i(x) % m] = 1

contains(x):
    return all(bits[hash_i(x) % m] == 1 for i in 1..K)
```

Returns:
- **Definitely not in set** — fast no.
- **Possibly in set** — true membership or false positive.

**Never false negative.** That's the magic.

### Math

For `m` bits, `n` elements, `k` hash functions:

```
False positive rate ≈ (1 - e^(-kn/m))^k
Optimal k ≈ (m/n) × ln 2
```

Practical numbers:
- For `n = 1M elements`, `m = 10M bits ≈ 1.25 MB`, `k = 7` → ~1% false positive.
- For 0.1% false positive: ~14 bits per element.

Exact set in a Java HashSet: ~64 bytes per element. Bloom filter: ~1.25 bytes. **50× less memory**.

### Use Cases

**1. Avoid expensive lookups.**

```
on read(key):
    if not bloom.contains(key):
        return None         # definitely not in DB, skip the disk read
    val = db.get(key)        # might be present, look up
    return val
```

Cassandra puts a bloom filter in front of every SSTable. Saves disk I/O on misses.

**2. CDN / cache existence checks.**

"Has user X ever visited this URL?" Bloom filter answers in memory; only fetch DB on possible hit.

**3. Distributed system membership.**

A node gossips a bloom filter of its keys; peers check before requesting.

**4. Bad URL / malware lists.**

Browsers carry a bloom filter of malicious URLs; on potential hit, do a real lookup.

**5. Network deduplication.**

"Have I sent this packet?" before retransmission.

### Limitations

- **No delete** (basic version). Setting a bit to 0 affects other items that hashed to it.
- **False positives, no false negatives.**
- **Sizing is fixed** at construction; resize means rebuild from source.

### Variants

| Variant | Adds |
|---------|------|
| **Counting Bloom filter** | Each "bit" is a small counter; supports delete |
| **Cuckoo filter** | Same FPR with smaller size + supports delete + better cache locality |
| **Scalable Bloom filter** | Adds new layers as it fills; supports growth |
| **Partitioned Bloom filter** | k disjoint regions, one per hash function |

In practice, Cuckoo filters are often a better modern choice — smaller, deletable, similar FPR.

### HyperLogLog (HLL)

Counts **unique items** ("cardinality") with sub-linear memory.

```
add(x):
    hash x
    look at leading zeros in hash
    update register based on max-leading-zeros seen

count_estimate():
    aggregate registers → estimate cardinality
```

With 16 KB of memory, estimates billions with ~1% error.

Used by:
- Distinct visitor counts.
- "Unique users this hour."
- Redis `PFADD` / `PFCOUNT`.
- Druid / Pinot / BigQuery `APPROX_COUNT_DISTINCT`.

Doesn't tell you *which* items; only how many distinct.

### Count-Min Sketch

Approximate **frequency** of items in a stream. "How many times has this key appeared?"

```
add(x): increment 4 different buckets (one per hash function)
count(x): return min across the 4 buckets
```

Properties:
- Overestimates (never under).
- Constant memory per bucket grid.
- Used for: heavy-hitters detection, top-K queries, traffic profiling.

Redis Stream `XADD` + custom; ClickHouse `topK`; some load balancers use it for hot-key detection.

### Tunable Sketches

t-digest, q-digest: approximate quantiles (percentile distributions) over streams. The right answer for high-cardinality timeseries percentile aggregation.

Used by Datadog, Elastic, Prometheus (`histogram_quantile`).

### Use-Case Cheat Sheet

| Question | Structure |
|---------|-----------|
| Is X in this set? | Bloom / Cuckoo filter |
| How many distinct items? | HyperLogLog |
| How many times has X appeared? | Count-Min Sketch |
| What are the heavy hitters? | Top-K via Count-Min |
| What's the p99 of a stream? | t-digest |
| What's the median? | t-digest or order statistics tree |

### Practical Sizing Example

100M unique events/hour, need to dedup:

- **Exact set (HashSet)**: 100M × 64 bytes = 6.4 GB per hour. Painful.
- **Bloom filter @ 1% FPR**: 100M × 1.25 bytes = 125 MB per hour. Comfortable.
- **Bloom filter @ 0.001% FPR**: 100M × 2.5 bytes = 250 MB. Still fine.

Pick FPR based on cost of a false positive in your use case.

### Common Mistakes

| Mistake | Fix |
|---------|-----|
| Treating Bloom filter as exact | False positives are guaranteed — design around them |
| Forgetting to size for growth | Plan capacity; use scalable BF or rebuild on threshold |
| One global filter for all keys | Hot key thrashes the bit array; shard or partition |
| Using HLL when you need exact counts | Approximate is the point; otherwise use a real counter |
| Bloom filter without a real-lookup fallback | A hit must be verified; otherwise it's not safe |

> [!NOTE]
> Sketches save memory in exchange for accuracy. Most analytics dashboards don't need 7-decimal precision; pick the right structure, free up gigabytes.

### Interview Follow-ups

- *"How does Cassandra use Bloom filters?"* — Per-SSTable filter; skip SSTables that definitely don't contain the key during reads.
- *"How accurate is HyperLogLog at low cardinality?"* — Bias-corrected variants (HLL++) handle small counts; vanilla HLL overestimates < 1000.
- *"Could you replace a Bloom filter with a cache?"* — A LRU cache of recent items is exact but limited. Bloom filter handles arbitrarily many items at the same memory.
