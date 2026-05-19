# Q: What are Virtual Threads (Project Loom), and when should you use them?

**Answer:**

Virtual threads (GA in Java 21) are **lightweight threads scheduled by the JVM** instead of the OS. They look like `Thread` and behave like `Thread` but cost orders of magnitude less. The point is to let you write **simple blocking code** at the throughput of an async framework.

### The Problem They Solve

Classic Java concurrency forces an unhappy trade:

- **Thread-per-request, blocking I/O** — readable, debuggable. But OS threads cost ~1–2 MB stack and a handful of microseconds to context-switch. A box runs out at ~few thousand threads.
- **Async / reactive** — scales to millions of concurrent operations. But: callback hell, stack traces lose context, debugging is misery, thread-locals don't work.

Virtual threads keep the thread-per-request mental model and remove the cost.

### How They Work

A virtual thread is a Java object scheduled onto a small pool of **carrier threads** (real OS threads, by default `ForkJoinPool` of size = CPU cores).

```
   ┌──────────────────────────────────────────────────┐
   │     10,000 virtual threads (Java heap objects)   │
   └─────────────────────┬────────────────────────────┘
                         │ scheduled onto
                         ▼
   ┌──────────────────────────────────────────────────┐
   │     ~N carrier threads (OS threads, N ≈ #CPU)    │
   └──────────────────────────────────────────────────┘
```

When a virtual thread hits a blocking call (e.g. `socket.read()`), the JVM **parks** it — saves its continuation off the carrier — and lets the carrier run another virtual thread. When I/O completes, the virtual thread is rescheduled. The OS thread never blocked.

### Creating Them

```java
// Direct
Thread.startVirtualThread(() -> handle(req));

// Or a builder
Thread.ofVirtual().name("handler-", 0).start(() -> ...);

// Executor — the recommended way
try (var exec = Executors.newVirtualThreadPerTaskExecutor()) {
    for (var req : requests) {
        exec.submit(() -> handle(req));
    }
}   // try-with-resources joins all tasks
```

There is **no pool** of virtual threads. You make one per task. Cheap.

### When Virtual Threads Help

They help when threads spend most of their time **waiting** on I/O — HTTP calls, DB queries, message brokers. A typical microservice handling requests via Servlet/Spring MVC fits perfectly.

```
Before (Tomcat, 200-thread pool):
   200 in-flight requests max, queue afterwards. p99 spikes under load.

After (virtual threads):
   10,000+ in-flight requests, no queueing.
```

Spring Boot 3.2+: `spring.threads.virtual.enabled=true`. Tomcat will hand each request to a virtual thread.

### When They Don't Help

- **CPU-bound work.** A virtual thread holds the carrier the whole time it's computing. Don't run a million Fibonacci tasks expecting magic — you're still limited by physical cores.
- **Code with native pinning** (see "Pinning" below).
- **Replacing a thread pool that's already right-sized for the work.** Pools sized for parallelism (not concurrency) are still correct.

### Pinning — the Main Gotcha

A virtual thread is **pinned** (can't unmount from its carrier) when:

1. It's inside a `synchronized` block.
2. It's executing a JNI/native call.

Pinned virtual threads block the carrier just like an OS thread. If many requests pin at the same time, you've recreated the bottleneck you tried to escape.

Diagnose:

```bash
-Djdk.tracePinnedThreads=full
```

Fix: replace `synchronized` with `ReentrantLock`, which is pin-aware in JDK 21. JDK 24+ removes the synchronized-pinning limitation (JEP 491).

### ThreadLocals — Behavior

Thread-locals still work, but they're a **gotcha at scale**:

- With 200 OS threads, 200 copies of each thread-local. Bounded.
- With 1,000,000 virtual threads, 1,000,000 copies. Memory pressure.

The replacement: `ScopedValue` (preview).

```java
private static final ScopedValue<User> USER = ScopedValue.newInstance();

ScopedValue.where(USER, currentUser).run(() -> handle(req));
```

`ScopedValue` is immutable and scoped to a call tree; no leak through pools.

### Virtual Threads vs Reactive Streams

| Aspect | Virtual threads | Reactive (Project Reactor, RxJava) |
|--------|----------------|------------------------------------|
| Style | Imperative, blocking | Async, functional |
| Stack trace | Full, useful | Often opaque |
| Backpressure | None built-in — manage with semaphores | First-class |
| Library compatibility | Any blocking JDBC, HTTP client works | Needs reactive client |
| Best for | I/O-heavy services | Streaming, fan-out, backpressure-critical |

For most CRUD services: virtual threads + blocking JDBC > reactive.

### Common Mistakes

| Mistake | Reality |
|---------|---------|
| Pooling virtual threads | Pointless — they're cheap. Use `newVirtualThreadPerTaskExecutor` |
| `synchronized (lock) { jdbc.call() }` | Pins the carrier across I/O. Use `ReentrantLock` |
| Using virtual threads for CPU work | No throughput gain; possibly worse due to scheduler overhead |
| Reading old advice about heap usage from a million Threads | Pre-Loom limits don't apply — virtual threads aren't OS threads |

> [!NOTE]
> Virtual threads aren't *faster* than OS threads at the unit level — they're cheaper at scale. You don't get them to handle one request faster; you get them to handle ten thousand at once.

### Interview Follow-ups

- *"Difference between virtual threads and green threads?"* — Conceptually similar (user-mode scheduled threads). Implementation: continuations on top of `ForkJoinPool`, full JVM integration, transparent yield on JDK blocking calls. Green threads in 1.x Java had no preemption and no multi-CPU support.
- *"What is a continuation?"* — The freeze-frame of a paused virtual thread (stack frames + locals + program counter). Exposed via `jdk.internal.vm.Continuation` for advanced use.
- *"How do they interact with Structured Concurrency?"* — JEP 462 / `StructuredTaskScope` provides try-with-resources style fan-out/fan-in across virtual threads. Same scope = same lifetime = automatic cancellation propagation.
