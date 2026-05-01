# Q: How does Garbage Collection work in Java?

**Answer:**

Garbage Collection (GC) automatically frees heap memory by reclaiming objects that are no longer reachable.

### How Objects Become Eligible for GC
An object is eligible when **no live thread** can reach it through any chain of references.

```java
Object a = new Object();  // Object created, referenced by 'a'
a = null;                  // Reference removed → object is now unreachable → GC eligible
```

### GC Process: Generational Collection

**1. Minor GC (Young Generation)**
- New objects are allocated in **Eden**.
- When Eden fills up, a minor GC runs.
- **Live objects** are copied to a **Survivor space** (S0 or S1).
- Dead objects are discarded (Eden is cleared).
- Objects that survive multiple minor GCs are **promoted** to Old Gen.

**2. Major GC / Full GC (Old Generation)**
- Triggered when Old Gen fills up.
- Much slower than minor GC (scans the entire heap).
- "Stop the world" — all application threads are paused.

### GC Algorithms

| Collector | Type | Pause | Best For |
|---|---|---|---|
| **Serial GC** | Single-threaded | Long STW | Small apps, single-core |
| **Parallel GC** (default < Java 9) | Multi-threaded | Medium STW | Batch processing, throughput |
| **G1 GC** (default Java 9+) | Region-based | Short STW | General purpose, balanced |
| **ZGC** (Java 15+) | Concurrent | < 1ms STW | Ultra-low latency |
| **Shenandoah** (Java 12+) | Concurrent | < 1ms STW | Low latency, Red Hat |

### G1 GC (Garbage-First)
The default collector since Java 9. Divides the heap into **equal-sized regions** and prioritizes collecting regions with the most garbage first.

```bash
java -XX:+UseG1GC -XX:MaxGCPauseMillis=200 MyApp
```

### ZGC (Z Garbage Collector)
Designed for **sub-millisecond pauses** regardless of heap size (supports up to 16 TB heaps).

```bash
java -XX:+UseZGC MyApp
```

### GC Tuning Tips
```bash
# Enable GC logging (Java 9+)
java -Xlog:gc*:file=gc.log:time,uptime,level,tags

# Set target pause time for G1
java -XX:MaxGCPauseMillis=100

# Monitor with jstat
jstat -gc <pid> 1000  # GC stats every 1 second
```

> [!TIP]
> In interviews, mentioning **G1 GC** (default, balanced) and **ZGC** (ultra-low latency) shows you understand modern Java. The key insight: "GC is a trade-off between **throughput** (total work done) and **latency** (pause duration). G1 balances both; ZGC minimizes latency at some throughput cost."
