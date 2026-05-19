# Q: How does async Rust work, and what does Tokio actually do?

**Answer:**

Rust's async is **futures + an executor**. The language gives you `async`/`await` and a `Future` trait; it does **not** give you a runtime. Tokio (or `async-std`, `smol`) supplies the executor and the async I/O primitives. Understanding the split is essential.

### What `async` Compiles To

```rust
async fn fetch(url: &str) -> Result<String> {
    let body = client.get(url).send().await?.text().await?;
    Ok(body)
}
```

The compiler rewrites this into a **state machine** implementing `Future`:

```
struct FetchFuture { state: enum { Start, AwaitSend, AwaitText, Done } }

impl Future for FetchFuture {
    fn poll(self: Pin<&mut Self>, cx: &mut Context) -> Poll<Output> {
        loop {
            match self.state {
                Start => /* kick off send() */,
                AwaitSend => match send_future.poll(cx) {
                    Pending => return Pending,
                    Ready(r) => /* advance to AwaitText */,
                },
                ...
            }
        }
    }
}
```

Each `.await` point becomes a state transition. The future is **lazy** — calling `fetch(url)` does nothing until something polls it.

### What `.await` Actually Does

```
1. Poll the inner future.
2. If Ready(val): return val, continue.
3. If Pending: store the Waker (from Context), return Pending up the stack.
```

The Waker is the channel back to the executor: "wake me when there's progress." This is *cooperative* multitasking — futures only yield at `.await` points.

### What Tokio Provides

```rust
#[tokio::main]
async fn main() {
    let v = tokio::spawn(fetch("https://...")).await;
}
```

Tokio supplies:

1. **A multi-threaded executor** (work-stealing thread pool of ~N=CPU workers) that polls spawned futures.
2. **A reactor** (`mio` + epoll/kqueue/IOCP) that wakes futures when sockets become ready.
3. **Async I/O types**: `TcpStream`, `File`, `tokio::sync::Mutex`, channels, timers.
4. **`tokio::spawn`** — submit a future to the executor's run queue.

```
                ┌──────────────────────────────┐
                │       Tokio Executor         │
                │   (work-stealing thread pool)│
                └──────────────┬───────────────┘
                               │ polls futures
                               ▼
                        ┌──────────────┐
                        │   Reactor    │  <- mio / epoll / kqueue / IOCP
                        └──────────────┘
                               ▲
                               │ wake when fd ready
                          OS kernel
```

### Spawning vs Awaiting

```rust
// Sequential — total time = A + B
let a = fetch("/a").await;
let b = fetch("/b").await;

// Concurrent — total time = max(A, B), within one task
let (a, b) = tokio::join!(fetch("/a"), fetch("/b"));

// Parallel across executor threads
let ja = tokio::spawn(fetch("/a"));
let jb = tokio::spawn(fetch("/b"));
let (a, b) = (ja.await.unwrap(), jb.await.unwrap());
```

- `.await` on a single future: sequential.
- `join!`/`try_join!`: concurrent **within one task** (no spawn).
- `spawn`: new task, runnable on any worker thread; truly parallel for CPU.

### The `Send` Bound

```rust
tokio::spawn(async move { ... });
//           ^^^^^^^^^ this future must be `Send`
```

If you hold a `!Send` type (like `Rc`, `RefCell`) across an `.await`, the future is `!Send` and won't compile in `spawn`. Common cause: holding a `std::sync::MutexGuard` across an `await`:

```rust
let g = std_mutex.lock().unwrap();
something(*g).await;        // ❌ guard held across await
```

Fix: use `tokio::sync::Mutex` (its guard is `Send`), or scope the lock:

```rust
let v = {
    let g = std_mutex.lock().unwrap();
    g.clone()
};
something(v).await;
```

### Blocking the Runtime

A long synchronous call inside an async task blocks the *entire worker thread*. If you happen to take all worker threads, the runtime stalls.

```rust
// ❌ blocks a worker
async fn handler() { std::thread::sleep(Duration::from_secs(5)); }

// ✅ yield via async sleep
async fn handler() { tokio::time::sleep(Duration::from_secs(5)).await; }

// ✅ offload CPU-bound work
let result = tokio::task::spawn_blocking(|| compute_heavy()).await?;
```

`spawn_blocking` puts the work on a separate, bigger thread pool (default 512). Use for blocking I/O (sync DB drivers, file ops) and CPU-bound code.

### Cancellation

Drop a future = cancel it. Code between `.await` points completes; the future then drops its state. No exceptions thrown.

```rust
let h = tokio::spawn(long_task());
h.abort();                            // sends cancellation
```

Implications:
- Use RAII (`Drop` impl) for cleanup, not "rescue blocks."
- Be aware that holding a lock across an `.await` and then being cancelled releases the lock when the guard drops — usually fine.

### Pinning

`Pin<&mut Self>` in `poll` exists because async state machines may contain **self-references** (an `&` into your own state). Moving such a value would dangle the reference. `Pin` prevents the move.

99% of async code uses `Box::pin` or `pin!()` and never thinks about it. You meet `Pin` head-on only when implementing `Future` by hand.

### Tokio Sync Primitives vs std

| std (sync) | tokio (async) | When |
|------------|--------------|------|
| `std::sync::Mutex` | `tokio::sync::Mutex` | Use tokio's if held across `.await` |
| `std::sync::mpsc` | `tokio::sync::mpsc` | tokio is async-aware |
| `std::sync::RwLock` | `tokio::sync::RwLock` | Same rule |
| - | `tokio::sync::oneshot` | Single-value send (e.g., reply channels) |
| - | `tokio::sync::Notify` | Like a condvar for async |
| - | `tokio::sync::Semaphore` | Bound concurrency |

### Common Mistakes

| Mistake | Fix |
|---------|-----|
| `for url in urls { fetch(url).await; }` when you wanted concurrency | `join_all`/`buffer_unordered` or spawn tasks |
| Holding `std::sync::Mutex` guard across `.await` | Use `tokio::sync::Mutex` or scope the lock |
| Heavy CPU work in `async fn` | `spawn_blocking` |
| Reading `bytes_read` from sync `Read::read` | Use `AsyncRead::read` |
| Using `tokio::spawn` for fire-and-forget without `.await` join handle | Errors silently dropped |

> [!NOTE]
> Rust async is *cooperative*. If you don't `.await`, you never yield. A CPU-bound async function is just a slow synchronous function.

### Interview Follow-ups

- *"Why doesn't Rust ship an executor?"* — Different programs want different schedulers: single-thread embedded, work-stealing servers, etc. Decoupling let runtimes (Tokio, embassy, monoio) compete.
- *"Difference between `async fn` and returning `impl Future`?"* — Functionally equivalent. The first is sugar; the second lets you express things like `+ Send` bounds without `async-trait`.
- *"What is `select!`?"* — Race multiple futures, run the branch corresponding to the first to complete. Cancellation-aware.
