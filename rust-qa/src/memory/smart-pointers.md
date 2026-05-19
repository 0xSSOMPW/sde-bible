# Q: When do you use `Box`, `Rc`, `Arc`, `RefCell`, and `Cell`?

**Answer:**

Rust ships with a family of **smart pointers** that each relax one specific rule of the standard ownership model in a controlled way. Picking the right one is the difference between idiomatic Rust and fighting the borrow checker.

### The Cheat Sheet

| Type | Ownership | Mutability | Thread-safe | Cost |
|------|-----------|-----------|------------|------|
| `Box<T>` | Single owner, heap | Inherited | Yes (if T is) | Heap alloc only |
| `Rc<T>` | Many owners, single-thread | Immutable | No | Non-atomic refcount |
| `Arc<T>` | Many owners, cross-thread | Immutable | Yes | Atomic refcount |
| `Cell<T>` | Single owner | Interior mutable (Copy values) | No (`!Sync`) | Zero |
| `RefCell<T>` | Single owner | Interior mutable (any) | No (`!Sync`) | Runtime borrow checks |
| `Mutex<T>` | Single owner (usually in `Arc`) | Interior mutable | Yes | Lock acquire |
| `RwLock<T>` | Same | Many readers OR one writer | Yes | Lock acquire |

### `Box<T>` — Heap Allocation

Use when:
- The value is too big for the stack (e.g., a large array).
- You need a known-size handle to an unknown-size value (recursive types).
- You want a trait object: `Box<dyn Trait>`.

```rust
enum List {
    Cons(i32, Box<List>),   // recursive — without Box, infinite size
    Nil,
}

let logger: Box<dyn Write> = Box::new(File::create("log")?);
```

`Box<T>` is just a heap pointer + automatic drop. No refcount.

### `Rc<T>` — Shared Single-Threaded Ownership

Use when:
- The compiler can't see that one of several borrowers will outlive the rest, so single-ownership doesn't fit.
- Single-threaded only (DOM, graphs, parent/child trees within one thread).

```rust
use std::rc::Rc;

let shared = Rc::new(String::from("config"));
let a = Rc::clone(&shared);
let b = Rc::clone(&shared);
// All three drop together; backing String freed when the last drops.
```

`Rc::clone` only bumps a counter — it does not copy the data.

### `Arc<T>` — Shared Cross-Thread Ownership

Same as `Rc` but uses atomic refcount operations. `Send + Sync` when `T: Send + Sync`.

```rust
use std::sync::Arc;
use std::thread;

let cfg = Arc::new(load_config());
for _ in 0..4 {
    let cfg = Arc::clone(&cfg);
    thread::spawn(move || serve(&cfg));
}
```

Cost: ~2× a non-atomic counter on x86. Don't use `Arc` "just in case" if you'll never share across threads — use `Rc`.

### `Cell<T>` — Interior Mutability for `Copy` Types

Lets you mutate through `&Cell<T>`. Only useful for `T: Copy` because the API is `get()`/`set()` (copies value in/out).

```rust
use std::cell::Cell;

struct Counter { count: Cell<u32> }

impl Counter {
    fn inc(&self) { self.count.set(self.count.get() + 1); }   // takes &self!
}
```

No runtime cost. Best fit for small numeric state inside a struct you want to expose immutably.

### `RefCell<T>` — Interior Mutability for Any Type

Defers borrow checking to runtime. `borrow()` returns `Ref<T>`, `borrow_mut()` returns `RefMut<T>`. Violations **panic**.

```rust
use std::cell::RefCell;

let data = RefCell::new(vec![1, 2, 3]);
data.borrow_mut().push(4);

// Panic at runtime:
let _r1 = data.borrow();
let _r2 = data.borrow_mut();   // BorrowMutError
```

Use sparingly. It's a last-resort tool for cases where the borrow checker is right but unhelpful (e.g., observer patterns where ownership and mutability genuinely interleave).

### Common Compositions

| Pattern | Means |
|---------|-------|
| `Box<dyn Trait>` | Single-owner trait object on the heap |
| `Rc<RefCell<T>>` | Multiple owners + mutable state (single-threaded) |
| `Arc<Mutex<T>>` | Multiple owners + mutable state across threads |
| `Arc<RwLock<T>>` | Same, optimized for many readers |
| `Arc<T>` (no lock) | Read-only shared state |
| `Rc<Vec<T>>` | Shared immutable list |

### Decision Flowchart

```
Need multiple owners?
├── No  → Box<T> (heap) or just T (stack)
└── Yes
    ├── Single thread?
    │   ├── Read-only: Rc<T>
    │   └── Need mutation: Rc<RefCell<T>>
    └── Multi-thread?
        ├── Read-only: Arc<T>
        ├── Mutation, simple: Arc<Mutex<T>>
        ├── Many readers: Arc<RwLock<T>>
        └── Atomics enough: Arc<AtomicX>
```

### Reference Cycles & `Weak`

`Rc`/`Arc` use **strong reference counts**. A cycle (A → B → A) never reaches 0 → leak.

```rust
use std::rc::{Rc, Weak};
use std::cell::RefCell;

struct Node {
    parent: RefCell<Weak<Node>>,    // upward = Weak (no cycle)
    children: RefCell<Vec<Rc<Node>>>,
}
```

`Weak<T>` holds a non-owning pointer; you upgrade it to `Rc<T>` if the value still exists (`weak.upgrade()` returns `Option<Rc<T>>`).

### Cost & Common Mistakes

| Mistake | Reality |
|---------|---------|
| Using `Arc` where `Rc` would do | Atomic ops are cheap but not free; matters in tight loops |
| `Arc<Mutex<T>>` held across `.await` | Deadlocks runtime — use `tokio::sync::Mutex` |
| `RefCell` instead of restructuring code | Often a sign that the design needs to split ownership differently |
| `Box::leak` to get `&'static` | Works for one-time init; never call it in a loop |
| Treating `Clone` of `Rc` like deep clone | It's a refcount bump. Use `(*rc).clone()` for deep |

> [!NOTE]
> A useful heuristic: if you're reaching for `Rc<RefCell<T>>`, ask whether you can rewrite using a typed index into a `Vec` (the "arena" pattern). Often cleaner and avoids interior mutability entirely.

### Interview Follow-ups

- *"Why isn't `Rc` `Sync`?"* — Its refcount uses non-atomic ops. Two threads cloning would race.
- *"How does `Box` differ from C++'s `unique_ptr`?"* — Conceptually identical. Rust enforces move-only at the type level; C++ does at convention.
- *"What's `Pin<Box<T>>`?"* — A `Box` whose contents are guaranteed not to be moved. Required for self-referential types (async generators).
