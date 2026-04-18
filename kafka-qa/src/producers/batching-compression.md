# Q: How do Batching and Compression work in Kafka producers?

**Answer:**

Batching and compression are Kafka's two most impactful performance optimizations on the producer side.

### Batching
Instead of sending each message individually over the network, the producer accumulates messages into **batches** and sends them together. This dramatically reduces network overhead.

**Key Configuration:**
```properties
batch.size=16384          # Max batch size in bytes (16 KB default)
linger.ms=5               # Max time to wait for a batch to fill before sending
```

**How it works:**
1. Producer receives a `send()` call.
2. The message is added to a batch buffer for the target partition.
3. The batch is sent when **either** `batch.size` is reached **or** `linger.ms` expires — whichever comes first.

```
linger.ms=0 (default):  Send immediately, tiny batches, many network calls.
linger.ms=5:            Wait up to 5ms to fill the batch, fewer calls, higher throughput.
linger.ms=100:          Wait up to 100ms, maximum batching, added latency.
```

> [!TIP]
> For high-throughput systems, set `linger.ms=5-20` and `batch.size=65536` (64KB) or higher. The small latency increase is usually negligible compared to the throughput gain.

### Compression
Kafka supports compressing message batches before sending them over the network. This reduces:
- **Network bandwidth** (often 50-80% reduction)
- **Disk storage** on brokers (compressed data stays compressed on disk)

**Producer Config:**
```properties
compression.type=snappy   # Options: none, gzip, snappy, lz4, zstd
```

**Comparison:**

| Algorithm | Speed | Ratio | CPU | Best For |
|---|---|---|---|---|
| `none` | Fastest | 1:1 | None | Low-volume topics |
| `snappy` | Fast | ~2:1 | Low | General-purpose (recommended) |
| `lz4` | Fast | ~2:1 | Low | High-throughput, balanced |
| `zstd` | Medium | ~3:1 | Medium | Best ratio, bandwidth-constrained |
| `gzip` | Slow | ~3:1 | High | Legacy, avoid in new systems |

### How They Work Together

```
Application: send(msg1), send(msg2), send(msg3), send(msg4)
                      │
        ┌─────────────▼──────────────┐
        │      Batch Accumulator     │
        │  [msg1, msg2, msg3, msg4]  │
        │   (wait for linger.ms or   │
        │    batch.size reached)     │
        └─────────────┬──────────────┘
                      │ compress batch
        ┌─────────────▼──────────────┐
        │   Compressed Batch         │
        │   (e.g., snappy: 60% smaller) │
        └─────────────┬──────────────┘
                      │ single network call
                      ▼
                   Broker
```

### Broker-Side Compression
The broker stores batches **in the same compressed format** they were received. It does NOT decompress and recompress. This means compression set by the producer extends to both network transfer AND disk storage — a double win.

> [!NOTE]
> If the broker's `compression.type` differs from the producer's, the broker will decompress and recompress, causing significant CPU overhead. It's best to let the producer control compression and set the broker to `compression.type=producer` (the default).
