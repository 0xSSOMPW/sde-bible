# Q: What is `ThreadLocal`? Why is it dangerous in thread pools?

**Answer:**

`ThreadLocal<T>` = per-thread variable. Each thread gets its own independent copy. Reads/writes never see other threads' values.

### Mental Model
Internally each `Thread` has a `ThreadLocalMap`. `ThreadLocal` is the key, value is per-thread.

### Common Use Cases
- **Per-request context**: user/tenant/correlation ID propagated through call stack.
- **Non-thread-safe utilities**: `SimpleDateFormat` (notoriously not thread-safe).
- **Transaction/session context** (Spring uses ThreadLocal heavily).

```java
private static final ThreadLocal<SimpleDateFormat> FMT =
    ThreadLocal.withInitial(() -> new SimpleDateFormat("yyyy-MM-dd"));

String today() { return FMT.get().format(new Date()); }
```

### Set / Get / Remove
```java
ThreadLocal<String> userId = new ThreadLocal<>();
userId.set("u-123");
userId.get();      // "u-123" — only on this thread
userId.remove();   // clear
```

### The Thread Pool Problem
Thread pools **reuse threads**. If you `set()` on a pooled thread and don't `remove()`, the next task on that thread inherits the stale value.

```java
ExecutorService pool = Executors.newFixedThreadPool(4);
ThreadLocal<String> tenant = new ThreadLocal<>();

pool.submit(() -> {
    tenant.set("acme");
    doWork();
    // forgot tenant.remove() ⚠️
});

pool.submit(() -> {
    System.out.println(tenant.get());  // "acme" — leaked from previous task!
});
```

### Memory Leak
`ThreadLocalMap` keys are weak references to the `ThreadLocal` object. Values are **strong references**. If the `ThreadLocal` becomes unreachable but the thread lives on (pool!), the value stays in the map forever until the slot is cleared lazily.

### Always Use Try/Finally
```java
public Object handle(Request r) {
    tenant.set(r.tenantId());
    try {
        return processor.run(r);
    } finally {
        tenant.remove();   // critical for pooled threads
    }
}
```

### Spring's MDC / RequestContextHolder
Spring/SLF4J's `MDC` and `RequestContextHolder` use `ThreadLocal` under the hood. Filter clears them after each request — that's why filters wrap chain calls in try/finally.

### `InheritableThreadLocal`
Child thread inherits parent's value at thread creation. Doesn't propagate to thread pool tasks (pool threads created at startup, not per-task).

### Modern Alternative: `ScopedValue` (Java 21+)
```java
final static ScopedValue<String> USER = ScopedValue.newInstance();

ScopedValue.where(USER, "u-123").run(() -> {
    // USER.get() == "u-123" only inside this run
});
// outside the run() — USER not bound
```
Immutable, no leak risk, designed for virtual threads.

### Virtual Threads + ThreadLocal
Virtual threads are cheap (millions). Each carries its own `ThreadLocalMap` → memory pressure. Prefer `ScopedValue` in virtual-thread code.
