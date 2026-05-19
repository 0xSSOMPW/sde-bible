# Q: How do you diagnose memory leaks in a JVM application?

**Answer:**

Java has garbage collection — but it also has memory leaks. The pattern is always the same: **a long-lived object holds references to objects that should have been collected**. Diagnosis is methodical, not magical.

### The Symptom

```
heap usage over time:
   ┌─────────────────────────────────────────────────┐
   │                                       ╱╲   ╱╲╱╲│
   │                              ╱╲     ╱╱  ╲╲╱   ╲│
   │                       ╱╲  ╱╱  ╲╲  ╱╱           │
   │                ╱╲  ╱╱  ╲╲╱     ╲╲╱             │
   │         ╱╲  ╱╱  ╲╲╱                            │
   │  ╱╲  ╱╱  ╲╲╱                                   │
   │╱╱  ╲╱                                          │
   └─────────────────────────────────────────────────┘
                    time →                eventually: OOM
```

A healthy app's heap sawtooths around a stable mean. A leak shows the mean drifting upward over hours/days, ending in `OutOfMemoryError: Java heap space`.

### Step 1: Confirm It's a Heap Leak

`OutOfMemoryError` has many flavors:

| Variant | Cause |
|---------|-------|
| `Java heap space` | Heap full of reachable objects — actual leak or undersized heap |
| `GC overhead limit exceeded` | GC running constantly but reclaiming <2% — same as above, earlier signal |
| `Metaspace` | Class metadata leak (classloader leak in web containers) |
| `Direct buffer memory` | `ByteBuffer.allocateDirect` not freed |
| `unable to create new native thread` | Thread leak — every `new Thread()` without shutdown |
| `Requested array size exceeds VM limit` | Single huge allocation (>2 GB) |

### Step 2: Capture a Heap Dump

In production, on OOM:

```bash
-XX:+HeapDumpOnOutOfMemoryError
-XX:HeapDumpPath=/var/log/heapdumps
```

Or live:

```bash
jcmd <pid> GC.heap_dump /tmp/dump.hprof
# Older:
jmap -dump:live,format=b,file=/tmp/dump.hprof <pid>
```

`live` runs a full GC first, dropping unreachable objects — what remains is what's leaking.

### Step 3: Analyze with Eclipse MAT or VisualVM

Eclipse MAT (Memory Analyzer Tool) is the workhorse:

1. Open `.hprof`.
2. Click **Leak Suspects** report — automated heuristic.
3. Inspect the **Dominator Tree** — objects that, if removed, free the most memory.
4. For a specific suspect, run **Path to GC Roots → exclude weak/soft/phantom references**.

The GC root path tells you *which long-lived object* is keeping your leaked instances alive.

```
java.lang.Thread (GC root)
  └── ConcurrentHashMap (the static cache)
        └── String "tenant-42-key"
              └── Order [12,345 instances]
```

The cache is the leak.

### Common Leak Patterns

**1. Unbounded static collection.**

```java
public class Cache {
    private static final Map<String, Object> CACHE = new HashMap<>();
    public static void put(String k, Object v) { CACHE.put(k, v); }
    // Never evicts. Lives forever.
}
```

Fix: `LinkedHashMap` access-ordered with `removeEldestEntry`, or Caffeine/Guava Cache with size or time bounds.

**2. Listener / observer not removed.**

```java
eventBus.register(this);   // every web request creates a new listener
// no unregister → eventBus pins this object forever
```

Fix: pair every register with unregister; use weak references; scope listeners properly.

**3. ThreadLocal in a pooled thread.**

```java
public static final ThreadLocal<UserContext> CTX = new ThreadLocal<>();

CTX.set(new UserContext(...));
// missing CTX.remove() → reusable Tomcat thread holds the context forever
```

In container thread pools, a `ThreadLocal` value can survive across requests. Always `CTX.remove()` in a `finally` block.

**4. ClassLoader leak in web container.**

A redeploy creates a new classloader for the webapp, but a static reference from a JDK-loaded class (e.g., a `Driver` registered in `DriverManager`, a `Logger`, a `ThreadLocal`) keeps the old classloader alive. Metaspace climbs after each redeploy.

Fix: cleanup hooks in `ServletContextListener.contextDestroyed`, deregister drivers, shutdown executors, clear ThreadLocals.

**5. Caches keyed by mutable objects.**

```java
Map<Order, BigDecimal> totals = new HashMap<>();
// Order's hashCode based on a field that changes — entry can never be retrieved or evicted
```

**6. Strings interned excessively.**

`String.intern()` puts strings in the StringTable, which lives in the heap (and metaspace pre-Java 8). Calling it on user-supplied data → leak.

### Step 4: Reproduce in Test

Once a suspect is identified, write a reproducer:

```java
@Test
void cacheDoesNotLeak() {
    for (int i = 0; i < 1_000_000; i++) {
        cache.put("k" + i, new byte[1024]);
    }
    System.gc();
    long heap = Runtime.getRuntime().totalMemory() - Runtime.getRuntime().freeMemory();
    assertThat(heap).isLessThan(100 * 1024 * 1024);
}
```

A failing test is the difference between "we think we fixed it" and "we know."

### Native Memory Tracking

`-XX:NativeMemoryTracking=summary` then:

```bash
jcmd <pid> VM.native_memory baseline
# run for a while
jcmd <pid> VM.native_memory summary.diff
```

Shows growth across Java Heap, Metaspace, Code Cache, Thread stacks, GC structures, Direct buffers. Useful when heap looks fine but RSS keeps growing.

### `MAT` Useful Queries

Dominator tree filtered:
- `SELECT * FROM java.util.HashMap WHERE @retainedHeapSize > 100000000`
- `SELECT * FROM java.lang.Thread WHERE @retainedHeapSize > 100000000`

Finding all ClassLoaders:
- Histogram → group by class → filter `ClassLoader`.

### Async Profiler for Allocations

Heap dumps show *what's there now*. To find *who's allocating*:

```bash
asprof -e alloc -d 60 -f alloc.html <pid>
```

Generates a flame graph by allocation site. Pairs well with heap dumps: dump shows the leak, profiler shows the code path producing leaked objects.

### Common Mistakes

| Mistake | Better |
|---------|--------|
| "GC will clean it up" | GC reclaims unreachable. Reachable garbage stays |
| Looking at `Runtime.totalMemory()` for leak detection | Use GC logs, post-GC heap-after numbers |
| Reading heap dump without `live` flag | Includes ephemeral garbage; harder to find the real culprit |
| Increasing `-Xmx` instead of finding the leak | Postpones the OOM, doesn't fix it |
| Assuming WeakReference fixes any leak | Only useful when GC-reachability really is the right policy |

> [!NOTE]
> A leak is reachable-but-useless memory. The fix is *always* to break the reference: clear the cache, remove the listener, drop the ThreadLocal, dispose the classloader.

### Interview Follow-ups

- *"Difference between strong, soft, weak, phantom references?"* — Strong: normal, GC won't reclaim. Soft: GC reclaims under memory pressure. Weak: GC reclaims on next cycle. Phantom: notified-only, used for cleanup hooks.
- *"How does G1 vs ZGC change diagnosis?"* — Doesn't change leak diagnosis. ZGC has lower pause times but doesn't reclaim reachable objects either.
- *"What's the difference between live heap and committed heap?"* — Live = objects currently reachable. Committed = OS-allocated heap memory. RSS can exceed committed due to off-heap, code cache, threads, metaspace.
