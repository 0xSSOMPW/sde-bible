# Q: How do `@Async` and `@Scheduled` work in Spring? Common gotchas.

**Answer:**

Both are **proxy-based** annotations. Spring intercepts the call and dispatches to a `TaskExecutor` (`@Async`) or `TaskScheduler` (`@Scheduled`).

### Enable
```java
@EnableAsync
@EnableScheduling
@Configuration class AsyncConfig { }
```

### `@Async` Basics
```java
@Service
class NotificationService {
    @Async
    public void send(Notification n) { httpClient.post(n); }

    @Async
    public CompletableFuture<Report> generate(Long userId) {
        Report r = build(userId);
        return CompletableFuture.completedFuture(r);
    }
}
```

Caller returns immediately. Method runs on the configured executor.

Return types:
- `void` — fire-and-forget.
- `Future<T>` / `CompletableFuture<T>` — caller can `.get()` / chain.
- `ListenableFuture<T>` — deprecated, prefer `CompletableFuture`.

### Configure Executor
```java
@Configuration
@EnableAsync
public class AsyncConfig implements AsyncConfigurer {
    @Override
    public Executor getAsyncExecutor() {
        ThreadPoolTaskExecutor ex = new ThreadPoolTaskExecutor();
        ex.setCorePoolSize(8);
        ex.setMaxPoolSize(32);
        ex.setQueueCapacity(500);
        ex.setThreadNamePrefix("async-");
        ex.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
        ex.initialize();
        return ex;
    }

    @Override
    public AsyncUncaughtExceptionHandler getAsyncUncaughtExceptionHandler() {
        return (ex, m, params) -> log.error("Async error in {}", m.getName(), ex);
    }
}
```

Multiple executors:
```java
@Bean("ioExecutor") Executor ioExecutor() { ... }
@Bean("cpuExecutor") Executor cpuExecutor() { ... }

@Async("ioExecutor")
public void download(URL u) { ... }
```

### `@Async` Pitfalls

**1. Self-invocation** — same as `@Cacheable`. Calling `this.async()` from within the same class bypasses the proxy → runs synchronously.

**2. Default executor**
Pre-Boot 3, default was `SimpleAsyncTaskExecutor` — **creates a new thread per call**. Disaster under load. Always configure explicitly.

**3. Exception propagation**
- `void` return → exception swallowed unless `AsyncUncaughtExceptionHandler` set.
- `Future` return → exception delivered via `Future.get()`.

**4. Transactions**
`@Async` runs in a different thread → loses transaction context from caller. Annotate the async method itself with `@Transactional` if needed (start a fresh tx).

**5. Security context**
ThreadLocal `SecurityContext` doesn't propagate. Use:
```java
@Bean
TaskDecorator securityDecorator() {
    return runnable -> {
        SecurityContext ctx = SecurityContextHolder.getContext();
        return () -> {
            try {
                SecurityContextHolder.setContext(ctx);
                runnable.run();
            } finally {
                SecurityContextHolder.clearContext();
            }
        };
    };
}
// Wire into ThreadPoolTaskExecutor#setTaskDecorator
```

### `@Scheduled` Basics
```java
@Component
class CleanupJob {
    @Scheduled(fixedRate = 60_000)              // every 60s, regardless of duration
    void cleanup() { ... }

    @Scheduled(fixedDelay = 60_000)             // 60s after previous run finishes
    void poll() { ... }

    @Scheduled(initialDelay = 5000, fixedRate = 30_000)
    void warmup() { ... }

    @Scheduled(cron = "0 0 2 * * *", zone = "UTC")   // 2am UTC daily
    void nightly() { ... }
}
```

### Cron Format (Spring Style)
```
sec  min  hour  day-of-month  month  day-of-week
0    0    2     *             *      *           → 2am every day
0    */5  *     *             *      *           → every 5 min
0    0    9     *             *      MON-FRI     → 9am weekdays
```
Spring also supports macros: `@hourly`, `@daily`, `@weekly`.

### Default Scheduler
Single-threaded! Long task blocks others. Configure:
```java
@Bean
TaskScheduler taskScheduler() {
    ThreadPoolTaskScheduler s = new ThreadPoolTaskScheduler();
    s.setPoolSize(10);
    s.setThreadNamePrefix("sched-");
    s.initialize();
    return s;
}
```
Or via property:
```yaml
spring.task.scheduling.pool.size: 10
```

### `@Scheduled` Pitfalls

**1. Multi-instance deployment**
Every instance fires the job. For a "run once cluster-wide" semantic, use:
- DB-backed lock (`ShedLock` — most common solution).
- Quartz with JDBC store.
- Leader election (e.g., via Kubernetes lease).

```java
@Scheduled(cron = "0 0 * * * *")
@SchedulerLock(name = "hourlyJob", lockAtLeastFor = "30s", lockAtMostFor = "10m")
void hourly() { ... }
```

**2. Method must be `void` and parameterless** (unless dynamic via `SchedulingConfigurer`).

**3. Exceptions** — uncaught exception kills next iteration of `fixedRate` jobs in some setups. Wrap in try/catch + log.

**4. Time zones** — server time vs UTC vs business time zone. Always specify `zone` in cron.

### Dynamic Schedules
```java
@Configuration
@EnableScheduling
public class DynamicSchedule implements SchedulingConfigurer {
    @Override
    public void configureTasks(ScheduledTaskRegistrar registrar) {
        registrar.addTriggerTask(
            () -> doWork(),
            ctx -> {
                String cron = config.getCronExpression();   // re-read each time
                return new CronTrigger(cron).nextExecution(ctx);
            });
    }
}
```

### `@Async` + `@Scheduled` Combined
```java
@Async
@Scheduled(fixedRate = 30_000)
public void refresh() { ... }
```
Decouples scheduling tick from work — scheduler thread isn't blocked.

### Modern Alternative — Virtual Threads
Boot 3.2+ on Java 21:
```yaml
spring.threads.virtual.enabled: true
```
Auto-uses virtual threads for `@Async`, request handling, and many other places. Reduces need to tune pool sizes.
