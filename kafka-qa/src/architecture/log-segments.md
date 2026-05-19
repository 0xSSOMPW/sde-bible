# Q: How is a Kafka partition stored on disk — segments, indexes, and the page cache trick?

**Answer:**

A Kafka partition is an **append-only sequence of bytes**, materialized on disk as a series of segment files plus index files. The on-disk layout is intentionally boring — that's why Kafka is fast.

### Per-Partition Directory Layout

```
/var/kafka/orders-0/
├── 00000000000000000000.log         <- segment data (messages)
├── 00000000000000000000.index       <- offset → file position
├── 00000000000000000000.timeindex   <- timestamp → offset
├── 00000000000019234567.log
├── 00000000000019234567.index
├── 00000000000019234567.timeindex
├── leader-epoch-checkpoint
└── partition.metadata
```

The number in the filename is the **base offset** — the offset of the first message in that segment.

### Segments

A partition is split into segments to make rolling and deletion cheap. Only the **active segment** (the one with the highest base offset) is being written. Closed segments are immutable.

A segment is rolled when:
- `segment.bytes` (default 1 GB) reached.
- `segment.ms` (default 7 days) elapsed.

Why segments matter:
- **Retention**: delete entire old segments — no defragmentation, no rewriting.
- **Compaction**: produce a new segment from old ones, swap atomically.
- **Tiered storage**: closed segments can be uploaded to S3.

### The Log File Format

Each segment `.log` is a sequence of **record batches**:

```
RecordBatch:
  baseOffset, batchLength
  partitionLeaderEpoch
  magic (record format version)
  CRC
  attributes (compression, transactional, control)
  lastOffsetDelta
  baseTimestamp, maxTimestamp
  producerId, producerEpoch, baseSequence
  records: [varint-encoded records]
```

Batches preserve producer-side batching all the way to disk. Compression (lz4, zstd) is per-batch. Consumers receive batches without re-encoding.

### `.index` — Sparse Offset Index

Maps offset → physical position. **Sparse**, not per-message:

```
relative_offset:int32   position:int32
        0                0
       128            4096       <- one entry per ~4 KB
       256            8192
       384           12288
```

Entry every `index.interval.bytes` (default 4096). To find offset N:
1. Binary search the sparse index for the largest entry ≤ N.
2. Scan forward in the `.log` from that file position.

Sparse index = small enough to mmap entirely. Linear scan from there = fast because of sequential disk reads.

### `.timeindex` — Time-to-Offset Index

```
timestamp:int64   offset:int64
```

Used for `--from-timestamp` consumer rewinds and retention by time. Also sparse.

### Why Kafka Is Fast — The Page Cache Story

Kafka writes to a normal file. The OS caches recently written pages in **page cache** (RAM). Consumers reading the tail of the log read from page cache, not disk.

```
Producer  ──►  write()  ──►  Page Cache  ──►  Disk (async by OS)
                                  │
                                  ▼
Consumer  ──◄── read()  ◄──  Page Cache (still warm)
```

Two consequences:
1. **Producer doesn't `fsync` per record.** It writes to the OS, returns. The OS flushes on its own schedule. Crash-safe via replication, not fsync.
2. **Tail-reading consumers hit RAM**, not disk. Throughput is bounded by network, not IOPS.

### Zero-Copy Send (`sendfile`)

When delivering to a consumer, Kafka uses `sendfile()` (Linux syscall) to copy bytes directly from the page cache to the socket buffer:

```
Without sendfile:
  page cache → user buffer → kernel socket buffer → NIC   (4 copies, 4 context switches)

With sendfile:
  page cache → NIC                                       (1 copy in kernel, 2 switches)
```

This works only if no transformation is needed — which is why Kafka stores compressed batches unchanged and never decompresses on the broker.

### Replication

Each partition has a leader and N replicas. Followers fetch from the leader exactly the same way consumers do. The leader tracks each replica's high-water mark. A record is *committed* once it's in all in-sync replicas (`min.insync.replicas`).

```
Leader:        offset 1000 (latest)
Follower A:    offset 998
Follower B:    offset 999
HW (committed): 998   ← min of in-sync followers
```

Consumers can only read up to the high-water mark.

### Log Compaction (vs Delete)

`cleanup.policy=delete` (default): retention by time or size, drop oldest segments.

`cleanup.policy=compact`: keep at least one record per key. The log cleaner periodically rewrites segments removing superseded records:

```
Before:    k1:v1, k2:v2, k1:v3, k3:v4, k1:v5
After:                          k2:v2, k3:v4, k1:v5
```

Useful for changelog topics, Kafka Streams state, KRaft metadata.

Both can be combined: `cleanup.policy=compact,delete`.

### Operational Implications

- **Disk choice**: sequential I/O dominates. SATA SSDs/NVMe are great. Spinning disk works if you have enough partitions to parallelize writes. RAID-10 over RAID-5/6.
- **No need to size the heap large.** Kafka uses ~4–8 GB heap; the rest of RAM is page cache. `-Xmx4g` is normal even on a 64 GB box.
- **fsync isn't durability** in Kafka. Replication is. Don't tune `flush.messages` / `flush.ms` aggressively — let the OS handle it.
- **Tail-read consumers ≠ historical-read consumers**: tail = RAM, historical = disk. A consumer that lags by hours pulls from disk and competes with the page cache.

### Examining a Segment

```bash
# Dump batches with the tool
kafka-dump-log.sh --files 00000000000019234567.log --print-data-log

# Output:
# offset: 19234567 position: 0 batchSize: 1234 ...
# offset: 19234568 timestamp: 1700000000000 key: "abc" payload: {...}
```

### Common Mistakes

| Mistake | Reality |
|---------|---------|
| "Kafka is fast because it skips disk" | It hits disk every write; OS caches it. Disk is sequential and cheap |
| Reducing segment.bytes for "faster" retention | More segments = more file handles, more index files, slower controller startup |
| Setting `flush.messages=1` for safety | Kills throughput, replication is the durability layer |
| Mixing compact and delete carelessly | Tombstones (null values) needed for compaction-delete |
| Mounting Kafka log dirs over NFS | Don't. Local block storage only |

> [!NOTE]
> The whole architecture is "let the OS be the database." Kafka stores bytes in order, lets Linux cache them, and ships them with `sendfile`. The cleverness is in not being clever.

### Interview Follow-ups

- *"Why doesn't Kafka use a B-tree like a database?"* — Append-only sequential writes are the access pattern. Sparse index + binary search is enough; trees would slow writes.
- *"Why does deleting old data not fragment the log?"* — Segments are entire files. Deletion = `unlink()`. No fragmentation possible.
- *"How does Tiered Storage interact with this?"* — Closed segments + their indexes are uploaded to remote (S3). Local copy is then deleted. Reads of old offsets transparently fetch from remote.
