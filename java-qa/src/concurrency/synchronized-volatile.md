# Q: What are `synchronized`, `volatile`, and Atomic classes?

**Answer:**

These are Java's three primary mechanisms for thread safety.

### `synchronized` — Mutual Exclusion (Locking)
Ensures only **one thread** can execute a block of code at a time by acquiring a monitor lock.

```java
// Synchronized method — locks on `this`
public synchronized void increment() {
    count++;
}

// Synchronized block — locks on a specific object
public void increment() {
    synchronized (this) {
        count++;
    }
}

// Static synchronized — locks on the Class object
public static synchronized void staticMethod() { }
```

**What `synchronized` guarantees:**
1. **Mutual exclusion** — only one thread enters the critical section.
2. **Visibility** — changes made inside the block are visible to other threads when the lock is released.

### `volatile` — Visibility (No Locking)
Ensures that reads and writes to a variable go directly to **main memory**, not a thread-local CPU cache. It guarantees **visibility** but NOT atomicity.

```java
private volatile boolean running = true;

// Thread 1
public void run() {
    while (running) { // Always reads from main memory
        doWork();
    }
}

// Thread 2
public void stop() {
    running = false; // Written to main memory immediately
}
```

**When to use `volatile`:**
- Simple flags (boolean stop/running flags).
- A variable written by one thread and read by many.
- NOT suitable for compound operations like `count++` (read + modify + write is not atomic).

### Atomic Classes — Lock-Free Thread Safety
The `java.util.concurrent.atomic` package provides classes that use **CAS (Compare-And-Swap)** CPU instructions for lock-free atomic operations.

```java
AtomicInteger counter = new AtomicInteger(0);

counter.incrementAndGet();     // Atomic: read + increment + write
counter.compareAndSet(5, 10);  // Atomic: if value is 5, set to 10
counter.addAndGet(3);          // Atomic: add 3 and return new value
```

### Comparison

| Feature | `synchronized` | `volatile` | Atomic |
|---|---|---|---|
| **Atomicity** | ✅ Yes (whole block) | ❌ No | ✅ Yes (single operation) |
| **Visibility** | ✅ Yes | ✅ Yes | ✅ Yes |
| **Blocking** | ✅ Yes (acquires lock) | ❌ No | ❌ No (CAS spin) |
| **Performance** | Slowest (lock contention) | Fast | Fast (no locks) |
| **Use case** | Complex multi-step operations | Simple flags | Counters, accumulators |

> [!CAUTION]
> `volatile` does NOT make `count++` thread-safe! `count++` is actually three operations: read, increment, write. Between the read and write, another thread can intervene. Use `AtomicInteger.incrementAndGet()` instead.
