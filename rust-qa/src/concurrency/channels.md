# Q: Channels in Rust — `mpsc`, `oneshot`, `broadcast`, `watch`. Which one when?

**Answer:**

Rust ecosystem ships several channel types. They look similar (send/recv) but have different semantics around senders, receivers, capacity, and replay. Picking the wrong one is the most common cause of "the message isn't being delivered" bugs.

### The Cheat Sheet

| Channel | Senders | Receivers | Capacity | Semantics |
|---------|---------|-----------|----------|-----------|
| `mpsc` | many | **one** | bounded or unbounded | FIFO queue, latest-receiver wins |
| `spsc` | one | one | bounded | Faster, single-producer |
| `oneshot` | one | one | 1 | Send once, receive once |
| `broadcast` | many | many | bounded ring | Each receiver gets every message |
| `watch` | one | many | 1 (latest only) | New subscribers see latest |
| `bounded` (crossbeam) | many | many | bounded | mpmc |

### `mpsc` — Multi-Producer, Single-Consumer

Tokio's `tokio::sync::mpsc` is the workhorse for async pipelines.

```rust
use tokio::sync::mpsc;

let (tx, mut rx) = mpsc::channel::<Event>(100);   // bounded

// Many producers
for _ in 0..4 {
    let tx = tx.clone();
    tokio::spawn(async move {
        tx.send(Event::new()).await.unwrap();
    });
}

// One consumer
while let Some(event) = rx.recv().await {
    process(event).await;
}
```

Bounded channels apply **backpressure** — `send()` awaits until there's space. This is the whole point: a slow consumer slows producers, not OOMs the queue.

Unbounded variant:

```rust
let (tx, mut rx) = mpsc::unbounded_channel();
tx.send(event).unwrap();             // non-async send, no backpressure
```

**Avoid unbounded for production.** They convert "consumer is slow" into "process runs out of memory and crashes."

### `oneshot` — Single-Use Reply

For request/response patterns where one task asks another for one value:

```rust
use tokio::sync::oneshot;

let (resp_tx, resp_rx) = oneshot::channel::<Reply>();
request_tx.send(Req { resp: resp_tx, ... }).await?;

let reply = resp_rx.await?;          // wait for one reply
```

The handler holds `resp_tx` and calls `resp_tx.send(reply)` exactly once. After that, both ends are consumed.

Use cases:
- Per-request reply channels.
- "Done" signals from spawned tasks.
- One-shot cancellation.

### `broadcast` — Pub/Sub

Every subscriber receives every message **after they subscribe**. Like a Linux ring buffer.

```rust
use tokio::sync::broadcast;

let (tx, _) = broadcast::channel::<Event>(32);

// Subscribers must call subscribe() and start receiving BEFORE messages send,
// or they'll miss them.
let mut rx1 = tx.subscribe();
let mut rx2 = tx.subscribe();

tokio::spawn(async move {
    tx.send(event).unwrap();
});

let _ = rx1.recv().await;
let _ = rx2.recv().await;
```

Important: capacity is per-subscriber. If a slow subscriber falls behind by more than the capacity, it gets a `RecvError::Lagged(N)` indicating it missed N messages — the channel doesn't block the producer for one slow consumer.

Use for: shutdown signals, config updates, event broadcasting.

### `watch` — Latest-Value Subject

A single-slot value with version tracking. Receivers always see the *latest* value, not the history.

```rust
use tokio::sync::watch;

let (tx, mut rx) = watch::channel(Config::default());

// Producer
tx.send(new_config).unwrap();

// Consumers
let cfg = rx.borrow().clone();        // current value, no wait
rx.changed().await?;                  // wait until next change
```

Use for: configuration updates, leader election state, "current value" semantics. Late subscribers see the current value, not the history.

### Crossbeam Channels

Outside Tokio (or for sync code):

```rust
use crossbeam_channel::{bounded, unbounded, select};

let (s, r) = bounded::<i32>(100);

// MPMC — many senders, many receivers
let r2 = r.clone();
let s2 = s.clone();
```

Crossbeam supports `select!` macro over multiple channels (Go-style):

```rust
select! {
    recv(r1) -> msg => handle(msg.unwrap()),
    recv(r2) -> msg => handle(msg.unwrap()),
    default(Duration::from_secs(1)) => println!("timeout"),
}
```

### `std::sync::mpsc` (Avoid)

The standard library's `mpsc` exists for historical reasons. It's slower than crossbeam, has fewer features, and isn't async-aware. Use `tokio::sync::mpsc` (async) or `crossbeam_channel` (sync).

### Backpressure Patterns

Bounded channels are the easy 80%. For more shape:

**1. Drop-when-full** (lose messages instead of blocking producer):

```rust
match tx.try_send(event) {
    Ok(_)                        => {},
    Err(TrySendError::Full(_))   => metrics.dropped.inc(),
    Err(TrySendError::Closed(_)) => break,
}
```

Used for telemetry where freshness > completeness.

**2. Coalesce / batch** (collect N before processing):

```rust
let mut batch = Vec::new();
loop {
    tokio::select! {
        Some(item) = rx.recv() => {
            batch.push(item);
            if batch.len() >= 100 { flush(&mut batch).await; }
        }
        _ = tokio::time::sleep(Duration::from_secs(1)) => {
            if !batch.is_empty() { flush(&mut batch).await; }
        }
    }
}
```

**3. `Semaphore` for concurrency cap** (not strictly a channel, but the same role):

```rust
let sem = Arc::new(Semaphore::new(10));
for url in urls {
    let permit = sem.clone().acquire_owned().await?;
    tokio::spawn(async move {
        fetch(url).await;
        drop(permit);
    });
}
```

### Closing Behavior

| Channel | Sender drops | Receiver drops |
|---------|-------------|----------------|
| `mpsc` | `recv()` returns `None` when all senders gone | `send()` returns `SendError` |
| `oneshot` | Sender drops → receiver gets `RecvError` | Receiver drops → sender's `send()` returns `Err(value)` |
| `broadcast` | All senders gone → `recv` returns `Closed` | Sender's `send` succeeds (no listeners) |
| `watch` | Sender drops → `changed()` returns `Err` | Sender's `send` returns `Err` |

This is how channels signal cancellation: just drop one end.

### Common Mistakes

| Mistake | Fix |
|---------|-----|
| Unbounded `mpsc` "for safety" | Memory grows unboundedly under load |
| `broadcast` subscribers created lazily after sends | New subscribers miss old messages |
| Cloning `oneshot::Sender` (it doesn't impl Clone) | Use `mpsc` if you need multiple senders |
| Holding a sender forever, expecting receiver to stop on `None` | Drop the sender when done so `recv()` returns `None` |
| `watch` for streaming events | It's latest-value only, not a stream |

> [!NOTE]
> Channel selection is a design question: what should happen when the receiver is slow? Block (mpsc bounded), drop (try_send), buffer-and-replay (broadcast), or "always show current" (watch). Pick the one that matches your domain semantics.

### Interview Follow-ups

- *"Why is `mpsc` only single-consumer?"* — Multi-consumer would need synchronization on every receive. `crossbeam_channel` is mpmc with that overhead; tokio's mpsc is faster by skipping it.
- *"How is async channel different from sync?"* — Async sends/receives integrate with the executor — they `.await` instead of blocking the thread.
- *"What's `Notify`?"* — Tokio's signal primitive — like a `Condvar` for async. Useful for "wake up the waiting task when something changes."
