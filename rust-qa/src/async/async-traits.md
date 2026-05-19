# Q: How do you write async methods in traits (and why was it hard)?

**Answer:**

For years, Rust's biggest async footgun was: **you can't put `async fn` in a trait**. Java's `interface Handler { CompletableFuture<R> handle(...); }` was simply not expressible in stable Rust. The `async_trait` crate filled the gap; native support arrived in Rust 1.75 (2023) and matured through 2024–2025.

### Why It Was Hard

```rust
trait Handler {
    async fn handle(&self, req: Request) -> Response;
}
```

The problem: `async fn` desugars to "returns `impl Future<Output = T>`". For a trait method, that means the return type is *anonymous and depends on the implementor*. Rust couldn't express that through a `dyn` boundary or even a generic trait until specific compiler work was done.

```
async fn handle(...) -> Response
   │
   ▼  desugars
fn handle(...) -> impl Future<Output = Response>
   │
   ▼  what's the size? what's the concrete type per impl?
   ?
```

### Pre-1.75: `async_trait` Macro

```rust
use async_trait::async_trait;

#[async_trait]
trait Handler {
    async fn handle(&self, req: Request) -> Response;
}

struct MyHandler;

#[async_trait]
impl Handler for MyHandler {
    async fn handle(&self, req: Request) -> Response { ... }
}
```

What the macro does: rewrites methods to return `Pin<Box<dyn Future<Output = T> + Send + 'async_trait>>`. Heap-allocates the future, type-erases it. Works on dyn-traits, but has runtime cost (Box allocation per call).

Variants for non-Send: `#[async_trait(?Send)]`.

### Native Async Trait Methods (Rust 1.75+)

```rust
trait Handler {
    async fn handle(&self, req: Request) -> Response;
}

impl Handler for MyHandler {
    async fn handle(&self, req: Request) -> Response { ... }
}
```

No macro. No Box allocation. The compiler generates an associated type for the future. Static dispatch works fully.

### The `dyn` Problem

Native async traits work great for **generic** dispatch:

```rust
fn run<H: Handler>(h: H) { ... }            // ✅ static dispatch, fast
```

But not (yet, easily) for `dyn`:

```rust
fn run(h: &dyn Handler) { ... }             // ❌ in many cases
```

`async fn` in a trait creates an unnameable associated future type, which `dyn` needs to know about. Workarounds:

**1. Return an explicit `impl Future` and box at the call site:**

```rust
trait Handler {
    fn handle(&self, req: Request) -> impl Future<Output = Response> + Send;
}

let boxed: Box<dyn Handler> = ...;
```

**2. Use `trait-variant` crate to generate both versions:**

```rust
#[trait_variant::make(SendHandler: Send)]
trait Handler {
    async fn handle(&self, req: Request) -> Response;
}
```

Generates `Handler` (any `Future`) and `SendHandler` (with `Send` bound).

**3. Keep using `async_trait` for `dyn`:**

```rust
#[async_trait]
trait DynHandler {
    async fn handle(&self, req: Request) -> Response;
}
```

Until native dynamic-dispatch async-trait stabilizes, `async_trait` remains the right call for plugin systems and trait objects.

### Send / Sync Bounds

Async functions return futures. For `tokio::spawn`, the future must be `Send`. Native async traits don't auto-bound `Send` — you must declare it where you need it.

```rust
trait Handler {
    fn handle(&self, req: Request)
        -> impl Future<Output = Response> + Send;
}
```

Or via `trait-variant`:

```rust
#[trait_variant::make(Send)]
trait Handler {
    async fn handle(&self, req: Request) -> Response;
}
```

### Lifetime Captures

A common bug:

```rust
trait Handler {
    async fn handle(&self, req: &Request) -> Response;
    //                          ^ captured in the future
}
```

The returned future implicitly captures `&self` *and* `&req`. The future's lifetime must be ≤ the shorter of the two. Compiler does this for you, but it bites at use sites:

```rust
fn run(h: impl Handler) {
    let req = Request::new();
    let fut = h.handle(&req);
    drop(req);            // ❌ fut borrows req
    block_on(fut);
}
```

### Common Patterns

**1. Service trait** (Tower-style):

```rust
trait Service<Req> {
    type Resp;
    type Err;
    async fn call(&mut self, req: Req) -> Result<Self::Resp, Self::Err>;
}
```

**2. Repository trait** with `dyn`:

```rust
#[async_trait]
trait UserRepo: Send + Sync {
    async fn find(&self, id: u64) -> Result<Option<User>>;
    async fn save(&self, u: &User) -> Result<()>;
}

let repo: Arc<dyn UserRepo> = Arc::new(PgUserRepo::new(pool));
```

**3. Plugin / extension trait** with native async:

```rust
trait Middleware {
    async fn before(&self, req: &mut Request);
    async fn after(&self, resp: &mut Response);
}
```

### Async Closures (Rust 2024 / 1.85+)

Closure-equivalent of native async traits:

```rust
let fetch = async |url: String| -> Result<String> {
    let r = reqwest::get(&url).await?;
    Ok(r.text().await?)
};

let body = fetch("https://...".into()).await?;
```

Implements `AsyncFn`, `AsyncFnMut`, `AsyncFnOnce` — mirrors `Fn`, `FnMut`, `FnOnce` for async.

### Cost Comparison

| Style | Per-call cost | dyn-friendly | When |
|-------|--------------|--------------|------|
| Native `async fn` in trait | Zero | Limited | Generic-bounded code |
| `async_trait` macro | Box alloc + virtual call | Yes | dyn traits, plugin systems |
| Manual `fn -> impl Future` | Zero | Manual boxing needed | Library APIs |
| `trait-variant` generated | Zero | Yes (Send variant) | Public APIs needing both |

### Common Mistakes

| Mistake | Fix |
|---------|-----|
| Forgetting `+ Send` on returned future when calling `tokio::spawn` | Use `trait-variant::make(Send)` or explicit `impl Future + Send` |
| Mixing `async_trait` and native — not the same signature | Pick one per trait |
| Holding non-Send across `.await` in a method that needs Send | Same rule as any async — refactor |
| Using `async_trait` for hot-path code | Box allocation per call; use native trait |

> [!NOTE]
> For new code: native async traits everywhere, `async_trait` only when you genuinely need `dyn`. For libraries with a public trait, `trait-variant` is worth the setup — gives consumers both `Send` and non-`Send` flavors.

### Interview Follow-ups

- *"Why is Tower's Service trait not async fn?"* — Predates native async traits. `Service::call` returns `Self::Future` explicitly; gives Tower flexibility for hand-rolled state machines and zero-alloc futures.
- *"Can you have `async` in a `dyn`-compatible trait yet?"* — Limited; you can declare it but `dyn Handler` itself has restrictions. Use `BoxFuture` return types for dyn-friendly traits today.
- *"What about RPITIT (return position impl trait in trait)?"* — That's exactly what native `async fn in trait` builds on. The same feature stabilized them.
