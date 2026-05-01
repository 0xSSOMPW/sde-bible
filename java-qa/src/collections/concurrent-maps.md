# Q: What is the difference between ConcurrentHashMap, Hashtable, and Collections.synchronizedMap?

**Answer:**

All three provide thread-safe Map implementations, but with very different performance characteristics.

### Hashtable (Legacy — Don't Use)
The original thread-safe Map. **Every method is `synchronized`**, meaning the entire map is locked for every operation.

```java
Hashtable<String, Integer> table = new Hashtable<>();
table.put("key", 1); // Locks the ENTIRE table
```

**Problems:** Only one thread can access the map at a time → massive bottleneck. Also, `null` keys/values are not allowed.

### Collections.synchronizedMap()
A wrapper that adds `synchronized` to every method of any Map. Same locking strategy as Hashtable.

```java
Map<String, Integer> map = Collections.synchronizedMap(new HashMap<>());
```

**Problem:** Still uses a **single lock** for the entire map. Must also manually synchronize iterations:
```java
synchronized (map) {  // Manual sync required!
    for (Map.Entry<String, Integer> e : map.entrySet()) { ... }
}
```

### ConcurrentHashMap (Use This!)
Uses **fine-grained locking** (segment-level in Java 7, node-level + CAS in Java 8+). Multiple threads can read and write to **different segments simultaneously**.

```java
ConcurrentHashMap<String, Integer> map = new ConcurrentHashMap<>();
map.put("key", 1); // Only locks the specific bucket, not the whole map
```

### Performance Comparison

| Feature | Hashtable | synchronizedMap | ConcurrentHashMap |
|---|---|---|---|
| **Locking** | Entire map | Entire map | Per-bucket / CAS |
| **Concurrent reads** | ❌ Blocked | ❌ Blocked | ✅ Lock-free |
| **Concurrent writes** | ❌ Sequential | ❌ Sequential | ✅ Parallel (different buckets) |
| **Null keys/values** | ❌ | ✅ (if wrapped map allows) | ❌ |
| **Iteration safety** | Fail-fast (throws CME) | Fail-fast (manual sync needed) | **Weakly consistent** (no CME) |
| **Performance** | Poor | Poor | Excellent |

### ConcurrentHashMap Atomic Operations

```java
// Atomic compute-if-absent (avoids check-then-act race)
map.computeIfAbsent("counter", k -> 0);

// Atomic merge
map.merge("counter", 1, Integer::sum);

// Atomic replace
map.replace("key", oldValue, newValue);
```

> [!TIP]
> The only reason to use `Collections.synchronizedMap` is if you need a thread-safe `TreeMap` or `LinkedHashMap` (since `ConcurrentHashMap` doesn't support ordering). For all other cases, **always use `ConcurrentHashMap`**.
