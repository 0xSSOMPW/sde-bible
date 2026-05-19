# Q: How do you choose and tune a JVM garbage collector (G1, ZGC, Parallel)?

**Answer:**

Modern JVMs ship four production GCs: **G1** (default), **ZGC**, **Shenandoah**, **Parallel**. Each makes different tradeoffs between **pause time**, **throughput**, and **memory overhead**. Picking one without understanding the tradeoff is how you ship "weird latency spikes."

### The Four GCs

| GC | Goal | Pause | Throughput | Heap size sweet spot |
|----|------|-------|-----------|---------------------|
| **Serial** | Tiny heaps | Stop-the-world | Low | < 100 MB |
| **Parallel** | Throughput | Long stop-the-world | Highest | Anything; batch jobs |
| **G1** | Balanced | < 200 ms (target) | Good | 1 GB – 32 GB |
| **ZGC** | Low pause | < 1 ms | Slightly lower | 1 GB – TB |
| **Shenandoah** | Low pause | < 10 ms | Similar to ZGC | OpenJDK distros |

Default since Java 9: **G1**.

### What a GC Actually Does

```
Heap = Young Generation (Eden + 2 Survivor spaces) + Old Generation

Allocate → Eden
Eden full → MINOR GC: copy live objects from Eden + Survivor-from to Survivor-to
After N survival cycles → promote to Old
Old fills up → MAJOR GC: collect Old (often more expensive)
```

GCs differ in how they walk the heap, when they pause, and how concurrent they are with the mutator (your code).

### G1 (Garbage-First)

Heap is split into ~2000 **regions** of equal size. G1 collects regions with the most garbage first (hence the name).

```
[E][E][O][O][H][O][S][E][E][O][O][O] ... 2048 regions
   E = Eden  S = Survivor  O = Old  H = Humongous (≥ region size)
```

Workflow:
- Young GC: short pause, collect Eden + Survivors.
- Concurrent marking: scan reachability in background.
- Mixed GC: collect Old regions selected from the marking phase.

Tuning knobs:

```
-Xms4g -Xmx4g                              # set both equal in containers
-XX:+UseG1GC                                # default in modern JDKs
-XX:MaxGCPauseMillis=200                    # target pause goal (G1 sizes regions to hit it)
-XX:G1HeapRegionSize=16m                    # explicit region size
-XX:InitiatingHeapOccupancyPercent=45       # start concurrent marking at 45% Old
-XX:G1NewSizePercent=20                     # min young gen %
-XX:G1MaxNewSizePercent=40                  # max young gen %
```

Hits most workloads' needs. Tune `MaxGCPauseMillis` and let G1 figure out the rest.

### ZGC

Concurrent collector. Almost all work happens *while application runs*. Pause times stay **sub-millisecond** regardless of heap size.

```
-XX:+UseZGC
-Xmx16g
-XX:+ZGenerational    # Java 21+: split young/old like G1, big throughput win
```

Use ZGC when:
- p99 latency matters more than throughput (trading, gaming, APIs with strict SLAs).
- Heaps are large (10s of GB to TB).
- You're willing to give up ~5–15% throughput for tiny pauses.

ZGC uses **colored pointers** (Linux x64 with 5-level page tables) and load barriers to relocate objects concurrently. Magic, but the overhead is per-load.

### Parallel GC

The throughput champion. Stop-the-world for both young and full collections, but parallelizes within each pause.

```
-XX:+UseParallelGC
-XX:ParallelGCThreads=8
```

Use for:
- Batch jobs where total wall time matters more than any individual pause.
- Map-reduce-style workloads.

### Choosing in 30 Seconds

```
heap > 32 GB?  → ZGC
latency SLA < 50 ms p99?  → ZGC or Shenandoah
batch throughput max?  → Parallel
default web service?  → G1
```

### Sizing the Heap

`-Xmx` should be set, not left to defaults. Common rule:

```
heap ≈ container_memory × 0.75
```

The other 25%: native memory (Metaspace, code cache, threads, direct buffers, off-heap caches), and Linux page cache.

For containers:

```
-XX:MaxRAMPercentage=75.0
-XX:InitialRAMPercentage=75.0
```

Modern JDKs auto-detect cgroup limits — verify with `-XshowSettings:vm`.

### `-Xms = -Xmx`

Set initial and max heap **equal** in long-running services. Why:
- Growing the heap is a stop-the-world operation in G1.
- Heap returns RAM to the OS lazily; not setting `Xms` won't make your container "use less."
- Surprises in monitoring (heap grows = looks like a leak).

### Useful Flags

```
-Xlog:gc*:file=gc.log:time,uptime,level,tags:filecount=10,filesize=10M

# Heap dump on OOM
-XX:+HeapDumpOnOutOfMemoryError
-XX:HeapDumpPath=/var/log/heapdumps

# Crash on OOM (let K8s restart)
-XX:+ExitOnOutOfMemoryError

# More descriptive errors
-XX:+ShowCodeDetailsInExceptionMessages

# String dedup (saves memory in apps with lots of identical strings)
-XX:+UseStringDeduplication
```

### Reading GC Logs

```
[12.345s][info][gc] GC(42) Pause Young (Normal) (G1 Evacuation Pause) 1024M->256M(2048M) 45.123ms
```

Decode:
- `Pause Young (Normal)`: minor collection.
- `1024M->256M(2048M)`: heap before → after (capacity).
- `45.123ms`: pause time.

Frequent young GCs with high `before→after` ratio? Big allocation rate; consider raising young gen.

`Full GC`s frequent? Old gen pressure; consider larger heap, ZGC, or fixing a memory leak.

### Tools

```bash
# Live GC stats
jstat -gcutil <pid> 1000

# Class histogram (top alloc classes)
jcmd <pid> GC.class_histogram | head -50

# Trigger heap dump
jcmd <pid> GC.heap_dump /tmp/dump.hprof
```

For deeper analysis: **GCViewer**, **GCEasy**, or load `gc.log` into Grafana via gc_log_exporter.

### Common Mistakes

| Mistake | Reality |
|---------|---------|
| Setting only `-Xmx`, not `-Xms` | Heap grows over time; metrics look like a leak; pause spikes during growth |
| ZGC for batch job | Throughput penalty for no benefit |
| Default GC for 64 GB heap | G1 struggles past ~32 GB; switch to ZGC |
| Tweaking 10+ flags without measuring | More knobs = more bugs. Start with `Xms/Xmx + MaxGCPauseMillis` |
| `-Xmx` equal to container memory | OOM-killed by kernel; leave 20–30% headroom |
| Disabling GC ("GC is the problem") | The allocations are the problem. Profile with `async-profiler -e alloc` |

### When To Care

GC tuning matters when:
- p99/p99.9 latency is bad and `gc.pause.max` is significant.
- Allocation rate is huge (multi-GB/s).
- Heap is large enough that G1's mixed GC stalls.

GC tuning does *not* fix:
- Memory leaks (reachable garbage).
- CPU-bound code paths.
- Excessive allocation patterns (fix the code, not the GC).

> [!NOTE]
> If GC takes >5% of CPU time *and* causes user-visible latency, tune. Otherwise leave it alone. The JVM defaults are excellent for most workloads.

### Interview Follow-ups

- *"What's `ZGenerational`?"* — Java 21 made ZGC generational (young + old). Massive throughput improvement at no pause cost. Almost always set it.
- *"Why does G1 have humongous regions?"* — Objects ≥ 50% of region size allocate directly into "humongous" regions. Avoids excessive copying.
- *"What's Epsilon GC?"* — A no-op GC. Allocates until OOM. Useful for measuring application allocation rate without GC noise, or for short-lived jobs.
