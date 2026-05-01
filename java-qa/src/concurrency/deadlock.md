# Q: What is a Deadlock? What about Livelock and Starvation?

**Answer:**

### Deadlock
Two or more threads are **permanently blocked**, each waiting for a lock held by the other.

```java
Object lockA = new Object();
Object lockB = new Object();

// Thread 1: acquires lockA, then waits for lockB
new Thread(() -> {
    synchronized (lockA) {
        Thread.sleep(100);
        synchronized (lockB) { System.out.println("Thread 1"); }
    }
}).start();

// Thread 2: acquires lockB, then waits for lockA
new Thread(() -> {
    synchronized (lockB) {
        Thread.sleep(100);
        synchronized (lockA) { System.out.println("Thread 2"); }
    }
}).start();

// 💀 DEADLOCK: Thread 1 holds lockA, waits for lockB.
//              Thread 2 holds lockB, waits for lockA.
//              Neither can proceed.
```

**Four conditions for deadlock (ALL must be present):**
1. **Mutual exclusion** — resources can't be shared.
2. **Hold and wait** — thread holds one lock while waiting for another.
3. **No preemption** — locks can't be forcibly taken away.
4. **Circular wait** — A waits for B, B waits for A.

**Prevention — consistent lock ordering:**
```java
// ✅ Both threads acquire locks in the SAME order
synchronized (lockA) {
    synchronized (lockB) { /* safe */ }
}
```

### Livelock
Threads are not blocked — they keep **actively responding to each other** but make no progress. Like two people in a hallway both stepping aside in the same direction.

```
Thread 1: "You go first" → releases lock
Thread 2: "No, you go first" → releases lock
Thread 1: "No really, you go first" → releases lock
... forever
```

**Solution:** Add randomized backoff or priority.

### Starvation
A thread is **perpetually denied access** to a resource because other higher-priority threads monopolize it.

**Example:** Using `synchronized` with no fairness guarantee. High-priority threads keep acquiring the lock, and a low-priority thread never gets its turn.

**Solution:** Use `ReentrantLock(true)` with **fair** ordering:
```java
ReentrantLock lock = new ReentrantLock(true); // Fair lock: FIFO ordering
```

### Summary

| Problem | Threads Blocked? | Making Progress? | Solution |
|---|---|---|---|
| **Deadlock** | ✅ Yes | ❌ No | Consistent lock ordering, timeout |
| **Livelock** | ❌ No (active) | ❌ No (repeating same actions) | Random backoff, priority |
| **Starvation** | ❌ Partially | ❌ One thread starved | Fair locks, priority adjustment |

> [!TIP]
> In production, use `jstack <pid>` or `Thread.getAllStackTraces()` to detect deadlocks. JVM thread dumps show "Found one Java-level deadlock" with the exact threads and locks involved.
