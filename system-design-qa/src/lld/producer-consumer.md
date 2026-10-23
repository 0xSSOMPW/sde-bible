# Q: Design a Producer-Consumer system (bounded buffer) - Low-Level Design.

**Answer:**

The most common concurrency LLD question. The core problem: producers generate work faster or slower than consumers process it; a **bounded buffer** decouples them, blocks producers when full (backpressure) and consumers when empty. The interviewer wants to see you reason about *wait/notify vs Lock+Condition vs Semaphore vs BlockingQueue*, shutdown, and correctness pitfalls (spurious wakeups, lost signals, deadlock).

### Clarifying Questions to Ask

- Bounded or unbounded? (Bounded - unbounded risks OOM; bounded gives backpressure.)
- Multiple producers / consumers? Single of each (enables lock-free)?
- Ordering guarantees (FIFO)? Priorities? Delayed items?
- What when full: block, drop newest/oldest, throw, or time out?
- Graceful shutdown: drain remaining items or stop immediately?
- Delivery semantics: at-most-once, at-least-once, exactly-once? In-process only, or distributed (Kafka/SQS)?
- Throughput/latency target (affects lock-free vs lock-based choice)?

### Requirements

**Functional**
- `put(item)` blocks while the buffer is full; `take()` blocks while empty.
- Thread-safe with N producers and M consumers; FIFO.
- Timeouts (`offer/poll` with timeout), non-blocking variants.
- Graceful shutdown without losing or hanging on items.

**Non-functional**
- No deadlock, no lost wakeups, no busy-wait.
- Bounded memory; fairness (no starvation) optional.
- High throughput; low contention.

### Core Entities / Class Diagram

```
 Producer(s) ──put──►  ┌────────────────────────┐ ──take──► Consumer(s)
 (Runnable)            │   BoundedBuffer<T>      │           (Runnable)
                       │  Object[] items         │
                       │  head, tail, count      │
                       │  lock + notFull/notEmpty│
                       └────────────────────────┘
                                 ▲
        «interface» BlockingBuffer<T> { put, take, offer(timeout), poll(timeout), size }
           ├─ WaitNotifyBuffer<T>        (synchronized, wait/notifyAll)
           ├─ LockConditionBuffer<T>     (ReentrantLock + 2 Conditions)
           ├─ SemaphoreBuffer<T>         (empty/full/mutex semaphores)
           ├─ LinkedBlockingQueue wrapper (production choice)
           └─ SpscRingBuffer<T>          (lock-free, single producer/consumer)
```

Classic invariants: `0 <= count <= capacity`; `count == items in [head, head+count)`; a producer sleeps only if `count == capacity`; a consumer only if `count == 0`.

### Design Patterns Used and Why

| Pattern | Where | Why |
|---|---|---|
| **Producer-Consumer** (the pattern itself) | Buffer between stages | Decouples rate and lifecycle of producers/consumers |
| **Monitor** | `synchronized` / Lock+Condition | Mutual exclusion + condition waiting |
| **Strategy** | `BlockingBuffer` implementations | Swap lock strategy by contention profile |
| **Poison Pill** (Sentinel) | Shutdown | Clean termination via the data path |
| **Thread Pool / Worker** | Consumers as workers | Executor = producer-consumer with a queue of Runnables |
| **Observer** (optional) | Metrics hooks | Queue depth, latency, throughput |

### Approach 1: `synchronized` + `wait/notifyAll`

```java
import java.util.*;
import java.util.concurrent.*;
import java.util.concurrent.atomic.*;
import java.util.concurrent.locks.*;

interface BlockingBuffer<T> {
    void put(T item) throws InterruptedException;
    T take() throws InterruptedException;
    boolean offer(T item, long timeout, TimeUnit unit) throws InterruptedException;
    T poll(long timeout, TimeUnit unit) throws InterruptedException;
    int size();
}

class WaitNotifyBuffer<T> implements BlockingBuffer<T> {
    private final Object[] items;
    private int head, tail, count;

    WaitNotifyBuffer(int capacity) { items = new Object[capacity]; }

    public synchronized void put(T item) throws InterruptedException {
        while (count == items.length) wait();          // WHILE, never IF: spurious wakeups + multiple waiters
        items[tail] = item;
        tail = (tail + 1) % items.length;
        count++;
        notifyAll();                                   // wake consumers (and producers; they re-check)
    }

    @SuppressWarnings("unchecked")
    public synchronized T take() throws InterruptedException {
        while (count == 0) wait();
        T item = (T) items[head];
        items[head] = null;                            // help GC
        head = (head + 1) % items.length;
        count--;
        notifyAll();
        return item;
    }

    public synchronized boolean offer(T item, long timeout, TimeUnit unit) throws InterruptedException {
        long deadline = System.nanoTime() + unit.toNanos(timeout);
        while (count == items.length) {
            long left = deadline - System.nanoTime();
            if (left <= 0) return false;
            TimeUnit.NANOSECONDS.timedWait(this, left);
        }
        items[tail] = item; tail = (tail + 1) % items.length; count++;
        notifyAll();
        return true;
    }

    @SuppressWarnings("unchecked")
    public synchronized T poll(long timeout, TimeUnit unit) throws InterruptedException {
        long deadline = System.nanoTime() + unit.toNanos(timeout);
        while (count == 0) {
            long left = deadline - System.nanoTime();
            if (left <= 0) return null;
            TimeUnit.NANOSECONDS.timedWait(this, left);
        }
        T item = (T) items[head]; items[head] = null; head = (head + 1) % items.length; count--;
        notifyAll();
        return item;
    }

    public synchronized int size() { return count; }
}
```

**Weakness**: a single wait set for both producers and consumers, so `notifyAll` wakes everyone (thundering herd) and `notify()` alone is unsafe (it might wake a producer when a consumer should wake -> lost wakeup/deadlock). Fine for small scale; Lock+Condition is better.

> [!NOTE]
> Always loop on the condition. A thread can wake without being notified (spurious wakeup), or be notified but find the condition consumed by another thread that got the monitor first. `if (cond) wait()` is a classic bug.

### Approach 2: `ReentrantLock` + two `Condition`s

```java
class LockConditionBuffer<T> implements BlockingBuffer<T> {
    private final Object[] items;
    private int head, tail, count;
    private final ReentrantLock lock;
    private final Condition notFull, notEmpty;

    LockConditionBuffer(int capacity) { this(capacity, false); }
    LockConditionBuffer(int capacity, boolean fair) {
        items = new Object[capacity];
        lock = new ReentrantLock(fair);                // fair = FIFO lock handoff; lower throughput
        notFull = lock.newCondition();
        notEmpty = lock.newCondition();
    }

    public void put(T item) throws InterruptedException {
        lock.lockInterruptibly();
        try {
            while (count == items.length) notFull.await();
            enqueue(item);
        } finally { lock.unlock(); }
    }

    public T take() throws InterruptedException {
        lock.lockInterruptibly();
        try {
            while (count == 0) notEmpty.await();
            return dequeue();
        } finally { lock.unlock(); }
    }

    public boolean offer(T item, long timeout, TimeUnit unit) throws InterruptedException {
        long nanos = unit.toNanos(timeout);
        lock.lockInterruptibly();
        try {
            while (count == items.length) {
                if (nanos <= 0) return false;
                nanos = notFull.awaitNanos(nanos);     // returns remaining time
            }
            enqueue(item);
            return true;
        } finally { lock.unlock(); }
    }

    public T poll(long timeout, TimeUnit unit) throws InterruptedException {
        long nanos = unit.toNanos(timeout);
        lock.lockInterruptibly();
        try {
            while (count == 0) {
                if (nanos <= 0) return null;
                nanos = notEmpty.awaitNanos(nanos);
            }
            return dequeue();
        } finally { lock.unlock(); }
    }

    private void enqueue(T item) {                     // caller holds lock
        items[tail] = item; tail = (tail + 1) % items.length; count++;
        notEmpty.signal();                             // exactly one consumer can proceed
    }
    @SuppressWarnings("unchecked")
    private T dequeue() {
        T item = (T) items[head]; items[head] = null; head = (head + 1) % items.length; count--;
        notFull.signal();                              // exactly one producer can proceed
        return item;
    }
    public int size() { lock.lock(); try { return count; } finally { lock.unlock(); } }
}
```

Two conditions allow targeted `signal()` (no thundering herd) - this is how `ArrayBlockingQueue` is built. Pitfall: `signal()` is safe only because each state change enables exactly one waiter of the other kind. With a variable number of items (batch put) use `signalAll`.

### Approach 3: Semaphores

```java
class SemaphoreBuffer<T> implements BlockingBuffer<T> {
    private final Object[] items;
    private int head, tail;
    private final Semaphore empty;                     // free slots
    private final Semaphore full = new Semaphore(0);   // filled slots
    private final Semaphore mutex = new Semaphore(1);  // protects indices (could be a Lock)

    SemaphoreBuffer(int capacity) { items = new Object[capacity]; empty = new Semaphore(capacity); }

    public void put(T item) throws InterruptedException {
        empty.acquire();                               // reserve a slot FIRST (outside mutex)
        mutex.acquire();
        try { items[tail] = item; tail = (tail + 1) % items.length; }
        finally { mutex.release(); }
        full.release();                                // publish one item
    }

    @SuppressWarnings("unchecked")
    public T take() throws InterruptedException {
        full.acquire();
        T item;
        mutex.acquire();
        try { item = (T) items[head]; items[head] = null; head = (head + 1) % items.length; }
        finally { mutex.release(); }
        empty.release();
        return item;
    }
    public boolean offer(T i, long t, TimeUnit u) throws InterruptedException {
        if (!empty.tryAcquire(t, u)) return false;
        mutex.acquire(); try { items[tail] = i; tail = (tail + 1) % items.length; } finally { mutex.release(); }
        full.release(); return true;
    }
    @SuppressWarnings("unchecked")
    public T poll(long t, TimeUnit u) throws InterruptedException {
        if (!full.tryAcquire(t, u)) return null;
        T item; mutex.acquire();
        try { item = (T) items[head]; items[head] = null; head = (head + 1) % items.length; } finally { mutex.release(); }
        empty.release(); return item;
    }
    public int size() { return full.availablePermits(); }
}
```

> [!NOTE]
> **Deadlock trap**: if `put` did `mutex.acquire()` *before* `empty.acquire()`, a producer could hold the mutex while blocked on a full buffer, and no consumer could get the mutex to free a slot. Always acquire the counting semaphore first, the mutex second, and release in reverse.

### Approach 4: `BlockingQueue` (what you use in production)

```java
BlockingQueue<Job> q = new ArrayBlockingQueue<>(100);       // array, one lock, bounded
// LinkedBlockingQueue: two locks (put lock/take lock) -> higher throughput, optionally bounded
// SynchronousQueue: capacity 0, direct hand-off (used by cached thread pool)
// PriorityBlockingQueue: unbounded heap by priority; DelayQueue: items available after a delay
// LinkedTransferQueue / ConcurrentLinkedQueue: lock-free CAS-based
```

### Graceful Shutdown: Poison Pill

```java
final class Task {
    static final Task POISON = new Task(-1, "STOP");           // identity-compared sentinel
    final int id; final String payload;
    Task(int id, String payload) { this.id = id; this.payload = payload; }
}

class Producer implements Runnable {
    private final BlockingBuffer<Task> buffer; private final int count; private final int idBase;
    Producer(BlockingBuffer<Task> b, int idBase, int count) { buffer = b; this.idBase = idBase; this.count = count; }
    public void run() {
        try {
            for (int i = 0; i < count; i++) buffer.put(new Task(idBase + i, "job-" + (idBase + i)));
        } catch (InterruptedException e) { Thread.currentThread().interrupt(); }
    }
}

class Consumer implements Runnable {
    private final BlockingBuffer<Task> buffer; private final String name; final AtomicInteger processed;
    Consumer(BlockingBuffer<Task> b, String name, AtomicInteger p) { buffer = b; this.name = name; processed = p; }
    public void run() {
        try {
            while (true) {
                Task t = buffer.take();
                if (t == Task.POISON) return;                  // one pill per consumer; do NOT re-put
                process(t);
                processed.incrementAndGet();
            }
        } catch (InterruptedException e) { Thread.currentThread().interrupt(); }
    }
    private void process(Task t) { /* simulate work */ }
}

public class Main {
    public static void main(String[] args) throws Exception {
        int producers = 3, consumers = 4, perProducer = 1000;
        BlockingBuffer<Task> buf = new LockConditionBuffer<>(16);
        AtomicInteger processed = new AtomicInteger();

        ExecutorService pc = Executors.newFixedThreadPool(producers);
        ExecutorService cc = Executors.newFixedThreadPool(consumers);
        for (int i = 0; i < consumers; i++) cc.submit(new Consumer(buf, "C" + i, processed));
        List<Future<?>> fs = new ArrayList<>();
        for (int i = 0; i < producers; i++) fs.add(pc.submit(new Producer(buf, i * perProducer, perProducer)));

        for (Future<?> f : fs) f.get();                        // wait until ALL producers finished
        for (int i = 0; i < consumers; i++) buf.put(Task.POISON);   // one pill per consumer, after all real work (FIFO)
        pc.shutdown(); cc.shutdown();
        cc.awaitTermination(10, TimeUnit.SECONDS);
        System.out.println("processed=" + processed.get() + " expected=" + producers * perProducer);
    }
}
```

Poison-pill rules: (1) send one per consumer, *after* all producers are done; (2) FIFO ensures real items are drained first; (3) with a non-FIFO queue (priority) give the pill the lowest priority; (4) alternatives: `volatile boolean running` + `poll(timeout)` loop, or `executor.shutdown()` / `shutdownNow()` + interrupt. Interrupt-based stops can drop in-flight items.

### Custom Bounded Buffer With Backpressure Policies

```java
enum OverflowPolicy { BLOCK, DROP_NEWEST, DROP_OLDEST, THROW }

class PolicyBuffer<T> {
    private final ArrayDeque<T> q = new ArrayDeque<>();
    private final int capacity; private final OverflowPolicy policy;
    private final ReentrantLock lock = new ReentrantLock();
    private final Condition notFull = lock.newCondition(), notEmpty = lock.newCondition();
    final AtomicLong dropped = new AtomicLong();

    PolicyBuffer(int capacity, OverflowPolicy p) { this.capacity = capacity; this.policy = p; }

    void put(T item) throws InterruptedException {
        lock.lockInterruptibly();
        try {
            while (q.size() == capacity) {
                switch (policy) {
                    case BLOCK -> notFull.await();
                    case DROP_NEWEST -> { dropped.incrementAndGet(); return; }
                    case DROP_OLDEST -> { q.pollFirst(); dropped.incrementAndGet(); }
                    case THROW -> throw new IllegalStateException("buffer full");
                }
            }
            q.addLast(item);
            notEmpty.signal();
        } finally { lock.unlock(); }
    }
    T take() throws InterruptedException {
        lock.lockInterruptibly();
        try {
            while (q.isEmpty()) notEmpty.await();
            T v = q.pollFirst(); notFull.signal(); return v;
        } finally { lock.unlock(); }
    }
}
```

Backpressure strategies: block producers (default; propagates slowness upstream), drop (telemetry, video frames), `CallerRunsPolicy` (producer runs the task itself, slowing itself), reject with error (HTTP 429/503), spill to disk.

### Lock-Free SPSC Ring Buffer (Disruptor idea)

With exactly **one producer and one consumer**, no lock is needed: the producer only writes `tail`, the consumer only writes `head`. Visibility through `volatile`/`VarHandle` release-acquire.

```java
class SpscRingBuffer<T> {
    private final Object[] buf;
    private final int mask;                              // capacity power of two: index = seq & mask (no modulo)
    private final AtomicLong head = new AtomicLong();    // next read (written by consumer only)
    private final AtomicLong tail = new AtomicLong();    // next write (written by producer only)

    SpscRingBuffer(int capacityPowerOfTwo) {
        if (Integer.bitCount(capacityPowerOfTwo) != 1) throw new IllegalArgumentException();
        buf = new Object[capacityPowerOfTwo]; mask = capacityPowerOfTwo - 1;
    }
    boolean offer(T item) {                              // producer thread only
        long t = tail.get();
        if (t - head.get() == buf.length) return false;  // full
        buf[(int) (t & mask)] = item;
        tail.lazySet(t + 1);                             // release store publishes the slot
        return true;
    }
    @SuppressWarnings("unchecked")
    T poll() {                                           // consumer thread only
        long h = head.get();
        if (h == tail.get()) return null;                // empty
        T item = (T) buf[(int) (h & mask)];
        buf[(int) (h & mask)] = null;
        head.lazySet(h + 1);
        return item;
    }
}
```

**LMAX Disruptor** generalizes this: pre-allocated ring of mutable events (no GC), sequence counters with **padding to avoid false sharing** (separate cache lines), multi-producer claiming via CAS on a sequence, and pluggable wait strategies (BusySpin, Yielding, Sleeping, Blocking). Reaches ~10-100M ops/s, p99 in microseconds. Trade-offs: busy-spin burns CPU; complexity; bounded power-of-two size.

MPMC lock-free needs per-slot sequence numbers (Vyukov's bounded MPMC queue) or CAS linked lists (Michael-Scott queue; `ConcurrentLinkedQueue`) - beware ABA (solved by GC in Java) and memory reclamation in C++.

### Relation to Thread Pools

`ThreadPoolExecutor` **is** a producer-consumer system: callers `execute()` (producers) put `Runnable`s into a `BlockingQueue`; worker threads (consumers) loop `queue.take()`.

```java
new ThreadPoolExecutor(
    core, max, 60, TimeUnit.SECONDS,
    new ArrayBlockingQueue<>(1000),                    // bounded queue = backpressure
    new ThreadPoolExecutor.CallerRunsPolicy());        // what to do when queue AND max threads are full
```

Queue choice changes pool behavior: unbounded `LinkedBlockingQueue` (fixed pool) never grows beyond core and can OOM; `SynchronousQueue` (cached pool) creates threads on demand and can explode thread count.

### Priority and Delay Queues

```java
record PriorityTask(int priority, long seq, Runnable r) implements Comparable<PriorityTask> {
    public int compareTo(PriorityTask o) {              // higher priority first; FIFO within a priority via seq
        int c = Integer.compare(o.priority, priority);
        return c != 0 ? c : Long.compare(seq, o.seq);
    }
}
PriorityBlockingQueue<PriorityTask> pq = new PriorityBlockingQueue<>();   // unbounded! cap via a Semaphore if needed

record DelayedTask(long runAtNanos, Runnable r) implements Delayed {
    public long getDelay(TimeUnit u) { return u.convert(runAtNanos - System.nanoTime(), TimeUnit.NANOSECONDS); }
    public int compareTo(Delayed o) { return Long.compare(getDelay(TimeUnit.NANOSECONDS), o.getDelay(TimeUnit.NANOSECONDS)); }
}
DelayQueue<DelayedTask> dq = new DelayQueue<>();       // take() blocks until the head's delay expires (basis of ScheduledThreadPoolExecutor)
```

Priority inversion / starvation: low-priority items may never run - use aging (boost priority with wait time) or separate queues with weighted fair scheduling.

### Concurrency Considerations (Hazards)

| Hazard | Cause | Prevention |
|---|---|---|
| **Spurious wakeup** | JVM/OS may wake a waiter | `while (!cond) await()` |
| **Lost wakeup** | Signal before wait, or `notify` wakes wrong kind | Hold lock while checking/waiting; use two Conditions; `notifyAll` with single monitor |
| **Deadlock** | Mutex acquired before counting semaphore; nested locks | Fixed lock ordering; acquire counting semaphore first |
| **Thundering herd** | `notifyAll` with many waiters | Separate Conditions + `signal` |
| **Starvation** | Unfair lock, priority queue | Fair lock, aging |
| **Visibility** | Plain fields read unsynchronized | Always access shared state under the lock or volatile/atomics |
| **Lost interrupt** | Swallowing `InterruptedException` | Restore the flag: `Thread.currentThread().interrupt()` |
| **Memory leak** | Not nulling dequeued slots | `items[head] = null` |
| **Shutdown hang** | Consumer blocked forever on `take()` | Poison pill, `poll(timeout)`, or interrupt |
| **Unbounded growth** | Using an unbounded queue | Bound capacity; monitor depth |
| **False sharing** | head/tail on the same cache line | `@Contended` / manual padding |

**Fairness**: `new ReentrantLock(true)` and `new Semaphore(n, true)` hand off in FIFO order; prevents starvation but ~10x lower throughput. Use only when starvation is observed.

**Multiple consumers need ordering?** FIFO dequeue order does not mean FIFO *completion* order. If per-key order matters (all events for user X in order), shard by key: N queues, hash(key) -> queue, one consumer per queue (this is exactly Kafka partitions).

### Extensibility

- Batching: `drainTo(collection, max)` to amortize lock cost; consumers process batches.
- Multiple stages (pipeline): queue between each stage; each stage its own pool (SEDA).
- Dynamic scaling of consumers by queue depth (autoscaler) with min/max bounds.
- Retry/DLQ: failed tasks re-enqueued with attempts count; after N failures to a dead-letter queue.
- Metrics: queue depth, enqueue/dequeue rate, wait time (put-latency), drop count.
- Replace buffer implementation via the `BlockingBuffer` interface (Strategy) without touching producers/consumers.

### Exactly-Once Handling (In-Process and Beyond)

- In-process, a queue hands each item to exactly one `take()` winner, so there are no duplicates *in the queue*; but a consumer crash mid-processing **loses** the item (at-most-once). To get at-least-once: take -> process -> ack; on crash, an in-flight registry with a visibility timeout re-enqueues.
- At-least-once + **idempotent consumer** (dedupe by item id, or upserts) = effectively exactly-once.
- Real exactly-once across systems requires transactional commit of "output + consumed offset" (Kafka transactions / outbox pattern), not magic.

### Scaling to Distributed / HLD (short)

```
Producers ──► Broker (Kafka topic: P partitions, replicated) ──► Consumer group (C <= P consumers)
                 │ key hash → partition (per-key ordering)        │ commit offset after processing
                 └─ retention (replay)                             └─ failed → retry topic → DLQ
```

| In-process concept | Distributed analogue |
|---|---|
| Bounded queue | Topic with retention + consumer lag monitoring; broker quotas |
| Blocking `put` (backpressure) | Producer `max.in.flight`, `buffer.memory` + blocking, rate limiting, 429s |
| Competing consumers | Consumer group; each partition to one consumer |
| Poison pill | Tombstone/end marker; or just graceful consumer-group rebalance |
| `take` + ack | Offset commit (Kafka), ack/visibility timeout (SQS), `ack` (RabbitMQ) |
| Shard by key | Partitions by key |
| Priority queue | Separate topics/queues per priority (Kafka lacks per-message priority) |
| Delay queue | SQS delay seconds, scheduled topic, Redis ZSET |

Differences: **Kafka** = durable log, consumers pull and track offsets, replayable, per-partition order, at-least-once by default (exactly-once via idempotent producer + transactions). **SQS** standard = at-least-once, no ordering, visibility timeout (message hidden while processing; reappears if not deleted), DLQ after maxReceiveCount; FIFO queues give ordering + dedupe (5 min window). **RabbitMQ** = push to consumers with prefetch (flow control). Backpressure in distributed form is usually *lag-based autoscaling* since producers cannot block on a remote queue.

### Testing Concurrent Code

- **Invariant tests**: N producers x M consumers, sum of produced == consumed, no duplicates (`ConcurrentHashMap.newKeySet().add` returns true), final size 0.
- **Stress**: many iterations, `CountDownLatch` start gate to maximize contention, tiny capacity (1-2) to force blocking.
- **Blocking behavior**: producer blocked on full buffer proves via `Thread.getState() == WAITING` or `Future.get(timeout)` throwing `TimeoutException`; then a `take()` releases it.
- **Interrupt/shutdown tests**: interrupt a blocked `take()` -> `InterruptedException`, flag handling.
- **Tools**: JCStress (memory-model litmus tests), Lincheck (linearizability), ThreadSanitizer (native), `jstack` for deadlock, JMH for throughput, Awaitility for async assertions. Avoid `Thread.sleep`-based tests.
- Run tests on multiple core counts; bugs hide on single cores.

```java
// Skeleton of a stress test
CountDownLatch start = new CountDownLatch(1);
Set<Integer> seen = ConcurrentHashMap.newKeySet();
// consumers: wait start.await(); loop take() until POISON; assert seen.add(id)
// producers: wait start.await(); put ids
start.countDown();
// join all; assert seen.size() == total
```

### Common Mistakes

| Mistake | Why wrong | Fix |
|---|---|---|
| `if (full) wait()` | Spurious wakeup / stolen condition | `while` loop |
| `notify()` with one monitor and mixed waiters | Wakes wrong thread -> deadlock | `notifyAll` or two Conditions |
| Busy-waiting `while(full) {}` / `Thread.sleep` polling | CPU waste, latency | Block on condition |
| Mutex before counting semaphore | Deadlock | Semaphore first, mutex second |
| Unbounded queue | OOM when consumers are slow | Bounded capacity |
| Swallowing `InterruptedException` | Thread cannot be stopped | Re-set interrupt flag, exit loop |
| One poison pill for N consumers | N-1 hang forever | One pill per consumer |
| Pill sent before producers finish | Items after the pill are lost | Join producers first |
| Re-putting the pill in each consumer (with bounded buffer) | Can block on full buffer | One pill per consumer is simpler |
| `HashMap`/`ArrayList` as the queue | Data races | Synchronize or concurrent structures |
| Holding the lock while processing | Serializes consumers | Take under lock, process outside |
| Not nulling slots | Memory leak | Null on dequeue |
| Assuming `size()` is exact under concurrency | Stale immediately | Use for metrics only |
| Using `ConcurrentLinkedQueue` and expecting blocking | It never blocks | `LinkedBlockingQueue` |

### Interview Follow-ups

- *"Why `while` and not `if` around `wait()`?"* — Spurious wakeups are allowed by the spec, and between notification and reacquiring the monitor another thread may change the state. The condition must be rechecked.
- *"`wait/notify` vs `Lock/Condition`?"* — A monitor has one wait set; `Condition`s give several, so producers and consumers wait separately and `signal` can target one thread. `Lock` also provides `tryLock`, interruptible/timed acquisition and optional fairness.
- *"Why can `notify()` be unsafe here?"* — With one wait set holding both producers and consumers, `notify` may wake a thread of the same kind that cannot proceed; the wake is consumed and everyone sleeps (lost wakeup).
- *"What is a semaphore solution's deadlock risk?"* — Acquiring the mutex before the empty/full semaphore: a blocked producer holds the mutex so consumers can never run. Order: counting semaphore, then mutex.
- *"Does `ArrayBlockingQueue` or `LinkedBlockingQueue` perform better?"* — `ABQ`: one lock for both ends, preallocated array, no per-item node garbage. `LBQ`: two locks (put/take) so producers and consumers do not contend, but allocates nodes. Under heavy two-sided contention LBQ usually wins; for low contention/predictable memory ABQ.
- *"What does `SynchronousQueue` do?"* — Zero capacity; a `put` waits for a `take` (direct hand-off). Used by `newCachedThreadPool`.
- *"How do you shut down consumers gracefully?"* — Stop producers, drain, send one poison pill per consumer (or use `shutdown()` + `awaitTermination`). `shutdownNow()` interrupts and returns unrun tasks.
- *"Poison pill vs interrupt vs volatile flag?"* — Pill flows through the data path and preserves FIFO draining; interrupt is immediate but may drop work; a volatile flag needs the consumer to wake up (use `poll(timeout)`).
- *"What about multiple producers and when to send pills?"* — Count producers finishing (`CountDownLatch`/`Future.get`) and send pills only after the last one; or have each producer send a pill and let consumers count N pills before exiting.
- *"How do you apply backpressure?"* — Bounded buffer so `put` blocks; or rejection/drop policies; or `CallerRunsPolicy`. Expose depth metrics and scale consumers.
- *"When would you drop instead of block?"* — Lossy telemetry, video frames, UI events: freshness over completeness. Choose DROP_OLDEST for "latest value wins", DROP_NEWEST to protect existing work.
- *"How do you guarantee fairness?"* — Fair lock/semaphore (FIFO waiter order); per-producer quotas or round-robin queues; priority aging.
- *"What is false sharing?"* — Two independently-written variables on one cache line cause coherence ping-pong between cores. Pad `head`/`tail` apart (`@Contended`).
- *"Explain the Disruptor."* — A pre-allocated ring buffer with sequence counters, no locks (CAS/volatile), cache-line padding, event objects reused, batch consumption, and pluggable wait strategies. Removes GC and lock contention; ~order of magnitude faster than `ArrayBlockingQueue`.
- *"How does the SPSC ring work without locks?"* — Each index has one writer; the producer publishes `tail` after writing the slot (release), the consumer reads `tail` (acquire) before reading the slot. Capacity power-of-two lets `& mask` replace modulo.
- *"What is the ABA problem?"* — In CAS-based structures, a value changes A->B->A and CAS wrongly succeeds. Java's GC mostly avoids it for node-based queues; use stamped references (`AtomicStampedReference`) or sequence numbers otherwise.
- *"How is a thread pool a producer-consumer?"* — `execute` enqueues `Runnable`; workers `take()` and run. Queue type + core/max sizes + rejection policy define its backpressure.
- *"What happens when the pool queue and max threads are full?"* — The `RejectedExecutionHandler` runs: Abort (throw), CallerRuns (backpressure), Discard, DiscardOldest.
- *"How would you implement a priority queue variant?"* — `PriorityBlockingQueue` with a comparator (priority, then sequence for FIFO ties); add a semaphore for bounding; use aging to avoid starvation.
- *"How to implement a delayed queue?"* — `DelayQueue` with `Delayed` items: `take()` blocks until the head expires. Custom version: heap + Condition with `awaitNanos(head delay)`, signaled when an earlier item is inserted.
- *"How do you ensure ordering with multiple consumers?"* — A single queue guarantees dequeue order, not processing order. Shard by key (hash -> queue -> one consumer) to preserve per-key order; this is partitioning.
- *"How do you get exactly-once?"* — Not from the queue alone. At-least-once delivery (ack after processing) plus idempotent consumers (dedupe by id / upsert), or transactional commit of results and offsets.
- *"What if a consumer crashes mid-task?"* — Without ack the task is lost. Use an in-flight set with visibility timeout, re-enqueue on timeout, retry count and a DLQ for poison messages.
- *"Distinguish poison *pill* from poison *message*."* — Pill = sentinel to stop consumers. Poison message = payload that always fails; it must be retried bounded times then parked in a DLQ.
- *"How do Kafka and SQS differ from `BlockingQueue`?"* — Kafka: persistent partitioned log, offsets, replay, ordering per partition, consumer groups. SQS: managed queue, visibility timeout, at-least-once (FIFO optional), no replay. `BlockingQueue`: in-memory, single JVM, lost on crash.
- *"How do you do backpressure across services?"* — Cannot block remotely: use bounded queues + 429/503 + retry-after, rate limiting, lag-based autoscaling, load shedding, circuit breakers, reactive streams `request(n)` demand signaling.
- *"How do you size the buffer and consumer count?"* — Little's law: `L = lambda * W`. CPU-bound: consumers ~ cores; IO-bound: cores x (1 + wait/compute). Buffer sized to absorb bursts (rate x burst duration), not to hide permanently slow consumers.
- *"How do you detect and diagnose deadlock?"* — `jstack`/`ThreadMXBean.findDeadlockedThreads()`; look for threads WAITING on conditions with nobody able to signal; use timeouts (`offer/poll` with timeout) and `tryLock` as defense.
- *"How do you test it?"* — Invariant stress tests with start-latch, tiny capacities, many iterations, uniqueness set; JCStress/Lincheck for memory-model and linearizability; blocking asserts with timeouts; no sleeps.
- *"Should consumers process under the lock?"* — No. Dequeue under lock, release, process. Otherwise consumers are serialized and a slow handler blocks producers.
- *"Why null out the slot on dequeue?"* — Otherwise the array retains a reference to a consumed object until overwritten (leak, GC pressure on large payloads).
- *"Is `size()` reliable?"* — Only as a snapshot; for control flow use blocking operations; for metrics it is fine.
- *"Virtual threads impact?"* — With Java 21 virtual threads, blocking `take()` is cheap (the carrier thread is released); you can have one virtual thread per task. Bounded buffers/semaphores still matter to limit concurrency to downstream resources. Avoid `synchronized` pinning on blocking calls - prefer `ReentrantLock`.
- *"Reactive alternative?"* — Reactive Streams (`Flow`, Project Reactor) formalize backpressure: subscriber requests n items; publisher never exceeds demand. No blocking threads.
- *"What if there is one producer and one consumer and latency is critical?"* — SPSC ring buffer (busy-spin or yielding wait); pin threads to cores; avoid allocation.
- *"What happens if a producer is interrupted while blocked in `put`?"* — `InterruptedException` thrown, item not added, interrupt flag cleared; restore the flag and stop or handle upstream (do not lose the item silently - retry or fail).
- *"Fairness in the consumer side?"* — Competing consumers on one queue naturally load-balance (fast ones take more); use prefetch=1 style behavior (RabbitMQ) so slow consumers do not hoard items.
