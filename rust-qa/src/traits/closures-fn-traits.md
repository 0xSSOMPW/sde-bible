# Q: `Fn`, `FnMut`, `FnOnce` — what's the difference?

**Answer:**

Closures in Rust implement one of three traits depending on **how they capture their environment**. The compiler picks the most permissive trait automatically. Understanding the hierarchy explains "cannot move out of captured variable" errors.

### The Hierarchy

```
        FnOnce        (called at least once, may consume captures)
          ▲
          │ super-trait
          │
         FnMut         (called many times, may mutate captures)
          ▲
          │ super-trait
          │
          Fn           (called many times, only borrows immutably)
```

Every `Fn` is also `FnMut` and `FnOnce`. Every `FnMut` is also `FnOnce`. Not vice versa.

### The Three Capture Modes

```rust
let s = String::from("hi");

// Fn — captures by &
let print = || println!("{}", s);
print(); print();                 // OK, called many times

// FnMut — captures by &mut
let mut v = vec![1, 2, 3];
let mut push = |x| v.push(x);
push(4); push(5);                 // OK, mutates

// FnOnce — captures by value (move)
let owner = move || drop(s);
owner();                          // OK
// owner();                       // ERROR: value moved
```

Compiler infers the minimum required. `move` keyword forces by-value capture (turns a `Fn` into a `Fn`-that-owns).

### When Each Is Required

Function signatures pick the looseness:

```rust
fn run_once<F: FnOnce()>(f: F)   { f(); }
fn run_many<F: FnMut()>(mut f: F) { f(); f(); }
fn run_shared<F: Fn()>(f: F)     { f(); f(); }
```

- `FnOnce` is the **most permissive bound** — accepts any closure, but you can only call it once.
- `Fn` is the **most restrictive bound** — but supports `&F` (shared) and calling from multiple places.

Rule of thumb: take the *least restrictive* bound your code needs.

### Function Pointers `fn`

`fn(T) -> U` is a separate type — a pointer to a free function, no environment. Coerces to all three traits.

```rust
fn double(x: i32) -> i32 { x * 2 }

let f: fn(i32) -> i32 = double;          // function pointer
let c: Box<dyn Fn(i32) -> i32> = Box::new(double);   // also fine
```

A closure with **no captures** also coerces to `fn`. Useful for callbacks across FFI:

```rust
extern "C" fn callback(x: i32) { ... }
unsafe { c_register(callback as *const u8); }
```

### Moving Captures: `move` Closures

```rust
let s = String::from("hi");
let closure = move || println!("{}", s);
// s no longer usable — moved into closure
```

Used heavily with `thread::spawn` and async tasks, where the closure outlives the caller's scope.

### Returning Closures

You cannot return a bare closure — its size is unknown.

```rust
fn make() -> impl Fn(i32) -> i32 {
    |x| x + 1
}

fn make_boxed() -> Box<dyn Fn(i32) -> i32> {
    Box::new(|x| x + 1)
}
```

- `impl Fn` — static dispatch, type known at compile time.
- `Box<dyn Fn>` — dynamic dispatch, can return different concrete closures from branches.

### Returning Different Closures From Branches

```rust
// ❌ ERROR — two different closure types
fn pick(neg: bool) -> impl Fn(i32) -> i32 {
    if neg { |x| -x } else { |x| x }
}

// ✅ works with dyn
fn pick(neg: bool) -> Box<dyn Fn(i32) -> i32> {
    if neg { Box::new(|x| -x) } else { Box::new(|x| x) }
}
```

Two `|...|` literals are two **distinct types** even if signatures match. `impl Trait` requires one concrete type.

### Capturing `&self` and Method Closures

```rust
struct Counter(u32);
impl Counter {
    fn incrementer(&mut self) -> impl FnMut() + '_ {
        move || self.0 += 1
    }
}
```

`'_` lifetime ties the closure to `&mut self`. The closure can't outlive the borrow.

### Common Errors

```rust
let mut v = vec![1];
let f = || v.push(2);
f();                                 // ERROR: closure was inferred Fn but needs FnMut
```

Fix: `let mut f = || v.push(2);` — `FnMut` closures need `mut`.

```rust
let s = String::from("hi");
let f = || drop(s);
f();
f();                                 // ERROR: FnOnce can only be called once
```

Fix: don't drop captured value, or rebuild it inside.

### Async Closures

Returning `async {}` from a closure gives `FnOnce -> impl Future`. Stable async closures (Rust 1.85+) help:

```rust
let process = async |x: u32| { fetch(x).await };
```

Before stable async closures, the common workaround: a closure returning `async move { ... }` block.

### Cheat Sheet

| Use case | Trait | Capture |
|---------|-------|---------|
| Pure transform, reused | `Fn` | `&T` |
| Stateful iterator | `FnMut` | `&mut T` |
| Spawned task / one-shot | `FnOnce` | `T` (by value) |
| Callback list | `Box<dyn Fn>` or `Box<dyn FnMut>` | varies |
| Cross-thread | `FnOnce + Send + 'static` | `move` |

### Common Mistakes

| Mistake | Fix |
|---------|-----|
| `F: Fn` when you need state mutation | `F: FnMut` and accept `mut f: F` |
| Returning closure without `Box`/`impl` | Use `impl Fn` for single concrete; `Box<dyn>` for runtime |
| Captures last too long, causing borrow errors | Add `move`, or restructure data ownership |
| Using `FnOnce` in a loop | Compile error — call it again? Wrap with `Option::take` if you must |

> [!NOTE]
> If a function takes a closure, prefer `FnOnce` first, relax to `FnMut`, then `Fn` only if you call repeatedly with shared access. Lower friction for callers.

### Interview Follow-ups

- *"What's the size of a closure?"* — Sum of its captured environment, padded. A no-capture closure is zero-sized.
- *"How is a closure compiled?"* — Anonymous struct holding captures + `impl Fn*` method body.
- *"`fn` vs `Fn` — what's the distinction?"* — Lowercase `fn` is a function pointer type. Uppercase `Fn` is a trait. Function pointers implement `Fn`.
