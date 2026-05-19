# Q: What are `Send` and `Sync`, and how does Rust prevent data races at compile time?

**Answer:**

`Send` and `Sync` are **auto-traits** that tell the compiler which types are safe to move or share across threads. They are the foundation of Rust's "fearless concurrency" promise: a data race is a compile error, not a heisenbug.

### Definitions

```rust
unsafe auto trait Send { }   // value can be MOVED to another thread
unsafe auto trait Sync { }   // value can be SHARED (&T) between threads
```

The exact relationship:

```
T: Sync ⇔ &T: Send
```

If you can send a `&T` to another thread, then `T` can be safely accessed from multiple threads simultaneously through shared references — that's what `Sync` means.

### Auto Traits

Both are implemented automatically for types whose fields are all `Send`/`Sync`. You almost never `impl Send` or `impl Sync` directly — it's `unsafe` to do so.

```rust
struct Position { x: f64, y: f64 }   // auto Send + Sync (only primitives)
struct WithRc(Rc<i32>)               // NOT Send, NOT Sync (because Rc isn't)
```

### What's Not `Send` or `Sync`

| Type | `Send`? | `Sync`? | Why |
|------|--------|--------|-----|
| `Rc<T>` | No | No | Non-atomic refcount; race on count = use-after-free |
| `Arc<T>` (where `T: Send+Sync`) | Yes | Yes | Atomic refcount |
| `Cell<T>` | depends on T | **No** | Interior mutability without synchronization |
| `RefCell<T>` | depends on T | **No** | Same as Cell, plus runtime borrow checks aren't thread-safe |
| `*const T`, `*mut T` | No | No | Raw pointers — you opt back in via `unsafe` |
| `MutexGuard<'a, T>` | No (on most OSes) | Yes | Some OS mutexes require unlock on the same thread |
| `Mutex<T>` (T: Send) | Yes | Yes | Mutex provides synchronization |

### How the Compiler Uses Them

`std::thread::spawn` is declared roughly:

```rust
fn spawn<F, T>(f: F) -> JoinHandle<T>
where
    F: FnOnce() -> T + Send + 'static,
    T: Send + 'static,
```

So the closure (and everything it captures) must be `Send`. Try to send an `Rc`:

```rust
let rc = Rc::new(5);
thread::spawn(move || println!("{}", rc));
// ERROR: `Rc<i32>` cannot be sent between threads safely
//        the trait `Send` is not implemented for `Rc<i32>`
```

The fix: use `Arc`.

### Sharing State: `Arc<Mutex<T>>`

```rust
use std::sync::{Arc, Mutex};
use std::thread;

let counter = Arc::new(Mutex::new(0));
let mut handles = vec![];

for _ in 0..10 {
    let c = Arc::clone(&counter);
    handles.push(thread::spawn(move || {
        *c.lock().unwrap() += 1;
    }));
}

for h in handles { h.join().unwrap(); }
assert_eq!(*counter.lock().unwrap(), 10);
```

Why both?
- `Arc<T>` provides **shared ownership** across threads (atomic refcount → `Send + Sync`).
- `Mutex<T>` provides **mutual exclusion** for interior mutation (the lock makes `&Mutex<T>` enough to mutate).

Together: `Arc<Mutex<T>>` is the standard pattern for "multiple owners, exclusive access at a time."

### `Sync` Without Mutation: Just `Arc<T>`

If `T` doesn't need mutation, you don't need a `Mutex`:

```rust
let config = Arc::new(load_config());
for _ in 0..4 {
    let c = Arc::clone(&config);
    thread::spawn(move || serve(&c));
}
```

Read-only `Arc<T>` is `Send + Sync` (when `T: Send + Sync`).

### Atomics: Sync Without a Lock

For primitive types:

```rust
use std::sync::atomic::{AtomicUsize, Ordering};

static COUNTER: AtomicUsize = AtomicUsize::new(0);
COUNTER.fetch_add(1, Ordering::Relaxed);
```

Atomics are `Sync` because their operations are atomic at the CPU level. Cheaper than `Mutex<usize>` for simple counters.

### Common Compile Errors and Cures

```rust
let counter = Rc::new(RefCell::new(0));
thread::spawn(move || *counter.borrow_mut() += 1);
//  ERROR: `Rc<RefCell<i32>>` cannot be sent
```

Cure: `Arc<Mutex<i32>>`.

```rust
let v = vec![Rc::new(1)];
v.par_iter().for_each(|r| println!("{}", r));
// ERROR: `Rc<i32>` is not Send
```

Cure: change inner type to `Arc<i32>`, or restructure to avoid sharing across threads.

### Manual Implementation (Rare, `unsafe`)

```rust
struct MyPtr(*mut u8);
// Auto-derived as !Send and !Sync because of *mut u8.

// Promise that our type is safe across threads (you'd better be right):
unsafe impl Send for MyPtr {}
unsafe impl Sync for MyPtr {}
```

You promise the compiler that your invariants hold. Wrong = undefined behavior.

### `Send` Without `Sync` — Real Case

`MutexGuard` is `!Send` on some platforms (because some pthread mutexes can only be unlocked by the locking thread). The lock itself (`Mutex<T>`) is `Send + Sync`, but the guard isn't.

`Cell<T>` is `Send` (if `T: Send`) — you can move ownership to another thread — but `!Sync` — you can't share a `&Cell<T>` because mutation has no synchronization.

### Decision Table

| Need | Use |
|------|-----|
| Read-only data shared across threads | `Arc<T>` |
| Mutable shared state, one writer at a time | `Arc<Mutex<T>>` |
| Many readers, few writers | `Arc<RwLock<T>>` |
| Single integer counter | `AtomicUsize` (static or in `Arc`) |
| Per-thread independent state | `thread_local!` |
| Async + shared mutation | `Arc<tokio::sync::Mutex<T>>` (do *not* hold `std::sync::Mutex` across `.await`) |
| Channel between threads | `std::sync::mpsc` / `crossbeam` / `tokio::sync::mpsc` |

> [!NOTE]
> Compile-time data race prevention is the single feature that justifies Rust's ownership complexity for many teams. If you're convinced a `Send`/`Sync` error is wrong, you're almost always missing an invariant the compiler is right about.

### Interview Follow-ups

- *"What is `'static` doing in `spawn`'s bound?"* — Threads outlive any caller's stack frame. The closure must not borrow short-lived data.
- *"Why is `Rc` faster than `Arc`?"* — Non-atomic refcount ops. Single-threaded contexts pay no atomic cost.
- *"How do channels avoid the `Send` requirement?"* — They don't. Sending a value over a channel requires `T: Send`. Channels are a *mechanism*; `Send` is the *policy*.
