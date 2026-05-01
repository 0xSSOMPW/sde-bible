# Q: How does ExecutorService and Thread Pools work?

**Answer:**

### Why Thread Pools?
Creating a new `Thread` for every task is expensive (OS thread creation, memory allocation). Thread pools **reuse a fixed set of threads** to execute many tasks.

### ExecutorService
The `ExecutorService` interface is the standard API for managing thread pools in Java.

```java
// Fixed-size pool: always 4 threads
ExecutorService pool = Executors.newFixedThreadPool(4);

// Cached pool: creates threads on-demand, reuses idle ones
ExecutorService pool = Executors.newCachedThreadPool();

// Single thread: tasks execute sequentially in one thread
ExecutorService pool = Executors.newSingleThreadExecutor();

// Scheduled: run tasks after a delay or periodically
ScheduledExecutorService pool = Executors.newScheduledThreadPool(2);
```

### Submitting Tasks

```java
ExecutorService pool = Executors.newFixedThreadPool(4);

// Fire-and-forget (Runnable)
pool.submit(() -> System.out.println("Task 1"));

// Get a result (Callable)
Future<Integer> future = pool.submit(() -> computeExpensiveThing());
Integer result = future.get(); // Blocks until done

// Shutdown
pool.shutdown();            // Finish current tasks, reject new ones
pool.shutdownNow();         // Interrupt running tasks immediately
pool.awaitTermination(5, TimeUnit.SECONDS); // Wait for completion
```

### ThreadPoolExecutor (Full Control)
The `Executors` factory methods are shortcuts. For production, configure `ThreadPoolExecutor` directly:

```java
ThreadPoolExecutor executor = new ThreadPoolExecutor(
    4,                // corePoolSize: minimum threads kept alive
    8,                // maxPoolSize: maximum threads allowed
    60, TimeUnit.SECONDS,  // keepAliveTime for idle threads above core
    new ArrayBlockingQueue<>(100),  // work queue capacity
    new ThreadPoolExecutor.CallerRunsPolicy()  // rejection policy
);
```

### Rejection Policies
When both the thread pool and work queue are full:

| Policy | Behavior |
|---|---|
| `AbortPolicy` (default) | Throws `RejectedExecutionException` |
| `CallerRunsPolicy` | The calling thread runs the task (backpressure) |
| `DiscardPolicy` | Silently discards the task |
| `DiscardOldestPolicy` | Discards the oldest queued task |

> [!WARNING]
> **Avoid `Executors.newCachedThreadPool()` in production** unless you're sure about your workload. It creates an **unbounded** number of threads. A sudden traffic spike can spawn thousands of threads, exhausting memory and crashing the JVM. Always use `ThreadPoolExecutor` with explicit bounds.
