# Q: Why is Kafka so fast?

**Answer:**

Kafka achieves extraordinary throughput (millions of messages/second) through several deliberate architectural decisions.

### 1. Sequential I/O (Append-Only Log)
Kafka writes messages to disk in a strictly **sequential, append-only** fashion. It never does random disk seeks.

Sequential disk writes are shockingly fast — often **600 MB/s+** on modern SSDs, compared to ~100 KB/s for random writes. This is because the OS can fully leverage disk write-ahead buffers and avoid head movement on HDDs.

### 2. Zero-Copy (sendfile)
When a consumer reads data, the **normal** path involves 4 copies:
```
Disk → Kernel Buffer → User Space → Socket Buffer → NIC
```

Kafka uses the Linux `sendfile()` system call to skip user space entirely:
```
Disk → Kernel Buffer → NIC  (zero-copy, 2 copies instead of 4)
```

This eliminates context switches and memory copies, reducing CPU usage and increasing throughput dramatically.

### 3. Page Cache (OS-Level Caching)
Kafka does NOT manage its own in-memory cache. Instead, it relies on the **OS page cache**. When data is written to disk, the OS caches it in RAM. When consumers read recent data, it's served directly from the page cache — essentially a free, automatically managed in-memory read cache.

```
Hot data (recent):  Served from OS page cache (RAM speed)
Cold data (old):    Read from disk (still fast due to sequential reads)
```

This is why Kafka's performance is often counter-intuitive: it writes to "disk" but reads from "memory."

### 4. Batching + Compression
Producers batch many messages together and optionally compress them. This means:
- Fewer network round trips
- Less disk I/O (one write for many messages)
- Smaller on-disk footprint

### 5. Partitioning (Horizontal Scaling)
Each partition is an independent log. Multiple partitions can be read/written in parallel across different brokers and consumers. Adding partitions and brokers scales throughput linearly.

### 6. No Per-Message Acknowledgment to Consumers
Unlike RabbitMQ (which tracks ACK per message), Kafka consumers simply track their offset position. There's no per-message bookkeeping on the broker side, which eliminates enormous overhead.

### Summary

| Technique | Benefit |
|---|---|
| Sequential I/O | Fast disk writes, no seeks |
| Zero-copy (sendfile) | Minimal CPU for data transfer |
| Page cache | Hot data served from RAM |
| Batching | Amortized network/disk overhead |
| Compression | Less bandwidth and storage |
| Partition parallelism | Linear horizontal scaling |
| Offset-based tracking | No per-message broker state |

> [!TIP]
> In interviews, the two killer points are **sequential I/O** and **zero-copy**. These are what fundamentally differentiate Kafka's performance from traditional message brokers that rely on random I/O and per-message routing.
