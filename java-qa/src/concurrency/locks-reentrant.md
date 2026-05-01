# Q: `synchronized` vs `ReentrantLock` vs `ReadWriteLock` vs `StampedLock`?

**Answer:**

| Lock | Reentrant | Try/Timeout | Fair | Read/Write split | Condition vars |
|---|---|---|---|---|---|
| `synchronized` | yes | no | no | no | one (`wait`/`notify`) |
| `ReentrantLock` | yes | yes | configurable | no | multiple |
| `ReentrantReadWriteLock` | yes | yes | configurable | yes | yes |
| `StampedLock` | **no** | yes | no | yes (+ optimistic) | no |

### `synchronized` (Intrinsic Lock)
```java
synchronized (lock) {
    // critical section
}
```
- Built into JVM, automatic release on exception/return.
- **No timeout**, no interruptibility, no try-lock.
- JVM optimizes (biased locking until JDK 15, lock coarsening, escape analysis).

### `ReentrantLock`
```java
private final ReentrantLock lock = new ReentrantLock();

public void op() {
    lock.lock();
    try {
        // critical section
    } finally {
        lock.unlock();   // ⚠️ must be in finally
    }
}
```

Capabilities:
```java
lock.tryLock()                              // non-blocking attempt
lock.tryLock(500, TimeUnit.MILLISECONDS)    // bounded wait
lock.lockInterruptibly()                    // respond to interrupt
new ReentrantLock(true)                     // fair (FIFO) — slower
```

### Conditions (Multiple Wait Queues)
```java
ReentrantLock lock = new ReentrantLock();
Condition notFull = lock.newCondition();
Condition notEmpty = lock.newCondition();

void put(E e) throws InterruptedException {
    lock.lock();
    try {
        while (queue.isFull()) notFull.await();
        queue.add(e);
        notEmpty.signal();
    } finally { lock.unlock(); }
}
```
With `synchronized` you only get one wait set per object.

### `ReentrantReadWriteLock`
Many readers OR one writer. Use when reads dominate writes.

```java
ReadWriteLock rw = new ReentrantReadWriteLock();

V get(K k) {
    rw.readLock().lock();
    try { return map.get(k); }
    finally { rw.readLock().unlock(); }
}

void put(K k, V v) {
    rw.writeLock().lock();
    try { map.put(k, v); }
    finally { rw.writeLock().unlock(); }
}
```

> [!WARNING]
> Read locks **don't block other reads** but **block writers**. Heavy read traffic can starve writers (use fair mode if needed).

### `StampedLock` (Java 8+)
Adds **optimistic read** — no lock acquisition for reads if uncontended.

```java
StampedLock sl = new StampedLock();

double distanceFromOrigin() {
    long stamp = sl.tryOptimisticRead();   // no lock
    double cx = x, cy = y;
    if (!sl.validate(stamp)) {              // writer happened during read?
        stamp = sl.readLock();              // fall back to real read lock
        try { cx = x; cy = y; }
        finally { sl.unlockRead(stamp); }
    }
    return Math.sqrt(cx*cx + cy*cy);
}
```

> [!IMPORTANT]
> `StampedLock` is **not reentrant**. Calling lock recursively → deadlock. No `Condition` support.

### Reentrancy
Same thread re-acquiring the lock it already holds:
```java
synchronized void a() { b(); }
synchronized void b() { /* same lock — OK */ }
```
All Java intrinsic + `ReentrantLock` support this. `StampedLock` does not.

### When to Pick What
- Default → `synchronized`. Simple, optimized, can't forget to unlock.
- Need timeout / interruptibility / fairness / multiple conditions → `ReentrantLock`.
- Read-heavy data structure → `ReentrantReadWriteLock` or `StampedLock`.
- Highest throughput, willing to handle complexity, no reentrancy → `StampedLock`.
- Avoid all of above → use `java.util.concurrent` data structures (`ConcurrentHashMap`, etc).
