# Q: How do you do back-of-envelope estimation in a system design interview?

**Answer:**

Most system design interviews expect numeric reasoning: storage in TB, requests per second, bandwidth in Gbps. The numbers don't have to be perfect — they have to be **defensible** and **order-of-magnitude correct**. Walk through assumptions out loud.

### Numbers Worth Memorizing

```
Time (orders of magnitude)
   L1 cache reference                           1 ns
   Branch mispredict                            5 ns
   L2 cache reference                           7 ns
   Mutex lock/unlock                           25 ns
   Main memory reference                      100 ns
   Compress 1KB with Zippy                  3,000 ns
   Send 2KB over 1 Gbps network            20,000 ns   = 20 µs
   SSD random read                        150,000 ns   = 150 µs
   Read 1MB sequentially from memory      250,000 ns   = 250 µs
   Round-trip within same datacenter      500,000 ns   = 500 µs
   Read 1MB sequentially from SSD       1,000,000 ns   = 1 ms
   Disk seek (HDD)                     10,000,000 ns   = 10 ms
   Read 1MB sequentially from HDD      20,000,000 ns   = 20 ms
   Send packet CA → Netherlands → CA  150,000,000 ns   = 150 ms
```

Source: Jeff Dean's classic table, updated for SSDs.

### Capacity Cheat Sheet

```
Server RAM:           commodity 64–256 GB, large 1–2 TB
Server cores:         8–96
Server NIC:           10–25 Gbps typical, 100 Gbps possible
SSD throughput:       3–7 GB/s sequential read, 100k+ IOPS random
HDD throughput:       150 MB/s sequential, 100 IOPS random
Postgres on SSD:      ~5,000–50,000 simple QPS per node depending on row size
Redis single node:    100k–1M ops/sec
Kafka broker:         100–500 MB/s sustained
S3:                   ~3,500 PUTs/sec/prefix, ~5,500 GETs/sec/prefix
```

### Quantities Worth Memorizing

```
Seconds in a day:     86,400 ≈ 10⁵
Seconds in a year:    31,536,000 ≈ 3 × 10⁷ (~ π × 10⁷)
1 KB:                 10³ bytes (or 2¹⁰; close enough)
1 MB:                 10⁶
1 GB:                 10⁹
1 TB:                 10¹²
1 PB:                 10¹⁵
1B users:             1,000,000,000 = 10⁹
1B/day:               ≈ 11,574 / sec
1B/year:              ≈ 32 / sec
```

### The Workflow

When the interviewer says "design X for Y users":

**1. Functional scope.**
Pick 2–4 core features. Don't try to design everything.

**2. Scale assumptions.**
DAU (daily active users) → QPS → storage → bandwidth.

**3. QPS.**
Reads vs writes. Peak ≈ 2–3 × average.

**4. Storage.**
Bytes per record × records per day × retention.

**5. Bandwidth.**
QPS × payload size.

### Worked Example: Twitter Read Path

> "Design Twitter for 200M DAU. 100 tweets/day read per user, 1 tweet/day written."

**QPS:**

```
Reads:  200M × 100 / 86,400 ≈ 230k QPS average
Writes: 200M × 1   / 86,400 ≈ 2.3k QPS average
Peak: 2–3× average → reads ~700k QPS, writes ~7k QPS
```

**Storage growth:**

```
Per tweet: 200 B (text) + 100 B (metadata) ≈ 300 B
Per day:   200M × 1 × 300 B = 60 GB
Per year:  60 GB × 365 ≈ 22 TB
5-year archive: ~110 TB (text only)
+ media (photo/video) — typically 10–100× larger
```

**Bandwidth:**

```
Read egress: 700k × 300 B ≈ 200 MB/s ≈ 1.6 Gbps
(plus media — much larger; usually served by CDN)
```

**Conclusion**: a few hundred servers for the read tier, fronted by CDN for media. One Postgres can't take 700k QPS — needs caching (Redis) and replication.

### Worked Example: URL Shortener

> "200M shortenings/day, 20B reads/day."

```
Write QPS:   200M / 86,400 ≈ 2,300/s
Read QPS:    20B  / 86,400 ≈ 230,000/s
Read/write ratio ≈ 100:1
```

**Storage:**

```
Per record:  ~500 B (long URL + short code + metadata + indexes)
Per day:     200M × 500 B = 100 GB
Per year:    100 GB × 365 ≈ 36 TB
5-year keep: ~180 TB
```

**Decision**: read tier dominated by cache. Origin DB can be smaller. Use Redis or in-memory LRU with TTL. Object storage for archive (rarely-accessed old links).

### Common Conversions

```
1 Gbps   = 125 MB/s
10 Gbps  = 1.25 GB/s
100 Gbps = 12.5 GB/s

Latency budgets
  Web page first paint:           < 1 s
  REST API p99:                   < 200 ms (good), < 1 s (acceptable)
  Cache hit:                      < 10 ms
  DB query (indexed):             < 5 ms in DC
  Cross-region call:              50–150 ms
```

### Heuristics

- **Daily → per-second**: divide by ~10⁵.
- **Peak factor**: 2–3× for global apps; 5–10× for spikes (Black Friday, viral content).
- **Read/write ratio**: most consumer apps are 10:1 to 1000:1 read-heavy.
- **Hot data**: 80/20 rule — 20% of keys account for 80% of accesses. Cache the 20%.
- **Index overhead**: ~30% on top of row size for typical Postgres tables.
- **Replication factor**: ×3 for storage cost in most replicated DBs.

### Validating With Constraints

Sanity-check by mapping numbers to known systems:

- "700k QPS" → larger than any single Postgres node; need sharding or caching.
- "100 TB" → fits in a single dedicated cluster, no need for object storage tier.
- "10 PB" → object storage is mandatory; pricing matters more than performance.

### Common Mistakes

| Mistake | Fix |
|---------|-----|
| Citing 1B users with no peak factor | Multiply by 2–3 for real-world spikes |
| Computing total bytes without retention | Multiply by retention years |
| Ignoring metadata + indexes + replication | Real storage = data × ~3–5 |
| Working in mixed units (Mbps vs MB/s) | Pick one; bytes/sec is usually cleaner |
| Forgetting media/blob payloads | Photos/videos dominate bandwidth |

> [!NOTE]
> Interviewers care more about *how you reason* than the exact answer. State assumptions, do arithmetic out loud, and revise when given a constraint. "I assumed 200M DAU, 100 reads each — does that match?" is a great prompt.

### Interview Follow-ups

- *"How would you handle 10× the load?"* — Identify the bottleneck from the numbers (writes, reads, storage, bandwidth) and propose specific scaling (sharding, caching, CDN).
- *"How much does this cost?"* — TB × $/TB for storage, QPS × $/req for managed services, Gbps × $/GB for egress.
- *"What's the latency budget end-to-end?"* — Decompose into client → CDN → LB → app → cache → DB; sum the worst-case to compare against SLA.
