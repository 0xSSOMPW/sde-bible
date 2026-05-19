# Q: How do you size a Kafka cluster — partitions, brokers, disk, network?

**Answer:**

Sizing is the most common "system design" Kafka question. Walk through it as a back-of-envelope calculation grounded in four numbers: **throughput, retention, replication factor, partition count**.

### Inputs You Need First

- **Peak ingress** (MB/s, not avg).
- **Retention** (hours or days).
- **Replication factor** (almost always 3).
- **Consumer fanout** (how many independent consumer groups read each byte).
- **Compression ratio** (typical 3–5x for JSON, 2x for already-compact binary).

### Step 1: Storage

```
disk_per_broker = (peak_ingress * retention * RF) / num_brokers
```

Example: 50 MB/s peak, 7 days retention, RF=3, 6 brokers.

```
total = 50 * 86400 * 7 * 3 = 90,720,000 MB ≈ 90 TB
per broker ≈ 15 TB
```

Add 30% headroom for compaction, indexes, OS, log roll, and operational margin → **~20 TB per broker**.

### Step 2: Network

Each byte produced is:
- Written once over network to leader.
- Replicated `RF-1` times.
- Read `fanout` times by consumers.

```
broker_egress = ingress_share * (RF - 1 + fanout)
```

Example: 50 MB/s peak, 3 consumer groups, RF=3, 6 brokers.

```
ingress per broker  = 50/6   ≈ 8.3 MB/s
egress per broker   = 8.3 * (2 + 3) ≈ 42 MB/s
```

At 10 GbE NIC = 1.25 GB/s. You're fine. At 1 GbE (125 MB/s) and a heavier fanout, you're not.

### Step 3: Partitions

Lower bound: enough partitions for peak consumer parallelism.

```
min_partitions >= max_consumers_in_one_group
```

If you ever want 24 parallel consumers, you need at least 24 partitions.

Upper bound considerations:
- Each partition has open file handles, a leader, replication threads, metadata.
- ZK-mode: keep total partitions per broker under ~4,000.
- KRaft: practical limits are much higher (10s of thousands per broker), but rebalance time still scales with partition count.
- More partitions = larger end-to-end latency floor (more batches to flush).

Rule of thumb:

```
partitions ≈ max(target_throughput / partition_throughput, consumer_parallelism)
```

Single-partition sustainable throughput depends on disk and replication but **10–30 MB/s** is a realistic planning number on commodity hardware.

### Step 4: Broker Count

Constraints:
- **Replicas per broker**: keep under ~4,000 (ZK) or ~10–20k (KRaft).
- **Disk per broker**: tied to retention. SSD/NVMe strongly preferred.
- **Failure domain**: RF=3 needs at least 3 brokers across at least 3 racks (set `broker.rack`).

For meaningful workloads, *minimum* useful cluster is **3 brokers**; production typically **5–6+** for headroom during node loss + maintenance.

### Worked Example

> *"We ingest 200k events/sec, avg 1 KB. Retention 14 days. 4 consumer groups. Plan it."*

- Ingress: 200,000 × 1 KB = 200 MB/s peak (assume = avg for simplicity).
- Storage: `200 * 86400 * 14 * 3 = 725 TB`. With 30% headroom → ~940 TB.
- 6 brokers → ~157 TB/broker. Too much. → **12 brokers, 80 TB/broker**, or compress (~3x typical) → **4 brokers viable, plan 6 for HA**.
- Partitions: enough for consumer parallelism. If each group runs 32 consumers, ≥ **32 partitions**, round up to **64** for headroom.
- Network/broker: `200/6 * (2 + 4) = 200 MB/s egress`. 10 GbE sufficient.

### Common Mistakes

| Mistake | Reality |
|---------|---------|
| Sizing for average, not peak | Lag balloons at peak; LinkedIn rule-of-thumb is plan for 2x peak |
| Ignoring fanout in network math | A 4-group cluster needs 4–5x the egress capacity of ingress |
| Too many tiny partitions "for safety" | Hurts latency, controller load, recovery time |
| RF=2 to save disk | One broker loss = under-replicated forever; use 3 minimum |

> [!NOTE]
> Always plan for `N+1` brokers — you must be able to lose a broker (or take one down for maintenance) without losing capacity.

### Interview Follow-ups

- *"What happens if you set RF higher than broker count?"* — Topic creation fails. RF ≤ broker count, always.
- *"Why not 1000 partitions per topic?"* — Producer batches per-partition; tiny partitions = tiny batches = bad throughput. Also, controller fail-over time scales with total partitions.
- *"How does KRaft change sizing?"* — Removes ZK as a metadata bottleneck, lets you scale partitions higher, faster controller fail-over. Storage/network math unchanged.
