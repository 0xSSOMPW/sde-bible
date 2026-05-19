# Q: `anyhow` vs `thiserror` — how do you design errors in real Rust applications?

**Answer:**

Idiomatic Rust error handling splits responsibility cleanly:

- **Libraries** use `thiserror` to define **specific, typed errors** consumers can match on.
- **Applications** use `anyhow` to **box and propagate any error**, attaching context as it bubbles up.

Both crates work with the standard `std::error::Error` trait. Neither replaces `Result`.

### The Library Side: `thiserror`

`thiserror` is a derive macro that generates `Display`, `Error`, and conversion impls for an error enum:

```rust
use thiserror::Error;

#[derive(Debug, Error)]
pub enum ApiError {
    #[error("invalid request: {0}")]
    BadRequest(String),

    #[error("user {id} not found")]
    NotFound { id: u64 },

    #[error("database error")]
    Database(#[from] sqlx::Error),

    #[error("io error")]
    Io(#[from] std::io::Error),
}
```

What you get:
- `Display` for each variant from the `#[error("...")]` format string.
- `From<sqlx::Error> for ApiError` via `#[from]` — `?` operator works.
- `source()` returns the wrapped error for the call chain.
- Zero runtime cost — it's all macro-expanded code.

The result: callers can `match err { ApiError::NotFound { id } => ... }`. Errors are part of your API.

### The Application Side: `anyhow`

`anyhow::Error` is a type-erased boxed error. You don't care which kind it is — you just want to propagate it up to `main` or an HTTP handler.

```rust
use anyhow::{Context, Result};

fn run() -> Result<()> {
    let cfg = load_config("./config.toml")
        .context("loading config")?;

    let db = connect(&cfg.db_url)
        .context("opening database")?;

    let user = db.fetch_user(42)
        .with_context(|| format!("fetching user {}", 42))?;

    Ok(())
}

fn main() -> Result<()> {
    run().context("application startup failed")?;
    Ok(())
}
```

Failure output:

```
Error: application startup failed

Caused by:
    0: fetching user 42
    1: connection refused
    2: io error: ECONNREFUSED
```

Each `.context()` adds a layer to the chain. The original error is preserved in `source()`. No `unwrap`, no `expect`.

### When to Reach for Which

| Situation | Use |
|-----------|-----|
| Public library, callers might match on errors | `thiserror` |
| Internal app code, errors only flow up to logs/handlers | `anyhow` |
| Binary's `main` | `anyhow::Result<()>` |
| Web handler returning HTTP status | Typed error → mapped to status |
| Quick scripts, prototypes | `anyhow` (or just `Result<T, Box<dyn Error>>`) |

You can mix freely. A library that returns `thiserror`-derived errors can be called from an `anyhow`-using app: `?` propagates because `anyhow::Error: From<E: Error>`.

### Why Not Just `Box<dyn Error>`?

`anyhow::Error` is essentially a typed-better `Box<dyn Error + Send + Sync + 'static>`. Differences:

- Smaller (single word in some configurations).
- Always carries a backtrace if `RUST_BACKTRACE=1`.
- `context` extension method for adding layers.
- Better error messages by default.

You can use `Box<dyn Error>` — anyhow is a strict upgrade for ergonomics.

### Backtraces

Both crates capture backtraces when `RUST_BACKTRACE=1` is set:

```
$ RUST_BACKTRACE=1 ./app
Error: failed to load config

Caused by:
    0: io error
    1: No such file or directory (os error 2)

Stack backtrace:
   0: anyhow::error::<impl anyhow::Error>::msg
   1: app::load_config
   ...
```

In production, capture with `RUST_BACKTRACE=1` and ship stack traces to your error tracker (Sentry, Rollbar). Cheap relative to value.

### Mapping to HTTP Status in Web Handlers

Define a typed app error, map at the boundary:

```rust
#[derive(Debug, Error)]
pub enum AppError {
    #[error("not found")]
    NotFound,
    #[error("validation: {0}")]
    Validation(String),
    #[error(transparent)]
    Internal(#[from] anyhow::Error),
}

impl IntoResponse for AppError {
    fn into_response(self) -> Response {
        let (status, msg) = match &self {
            AppError::NotFound       => (StatusCode::NOT_FOUND, "not found".into()),
            AppError::Validation(m)  => (StatusCode::BAD_REQUEST, m.clone()),
            AppError::Internal(e)    => {
                tracing::error!("{:#}", e);    // alternate format prints chain
                (StatusCode::INTERNAL_SERVER_ERROR, "internal error".into())
            }
        };
        (status, msg).into_response()
    }
}
```

Inside the handler:

```rust
async fn get_user(Path(id): Path<u64>) -> Result<Json<User>, AppError> {
    let user = db.find_user(id).await?
        .ok_or(AppError::NotFound)?;
    Ok(Json(user))
}
```

`thiserror` for the boundary, `anyhow` internally via `Internal(#[from] anyhow::Error)`.

### Adding Context — `with_context` vs `context`

```rust
do_thing().context("static description")?;             // string is alloc'd lazily
do_thing().with_context(|| format!("id {}", id))?;     // closure only runs on Err
```

Use `with_context` for messages with formatting — closure avoids the allocation on the happy path.

### Common Mistakes

| Mistake | Fix |
|---------|-----|
| `unwrap()` in production code | Use `?` everywhere; propagate to main |
| `String` errors (`Err("bad".into())`) | Anonymous strings lose structure — wrap in a typed error |
| `Box<dyn Error>` in a public lib | Callers can't match on it; use `thiserror` enum |
| Manually implementing `Display` and `Error` | Use `thiserror` — it's literally fewer lines |
| Multiple wraps without context | Add `.context(...)` at each layer for better chains |
| Converting away the source via `to_string()` | Lose the chain; just use `?` and `#[from]` |

### Anyhow Macros

```rust
// Equivalent to Err(anyhow!("..."))
anyhow::bail!("user {} not found", id);

// Build an Err without returning
let e: anyhow::Error = anyhow!("oops");

// Assert a precondition
anyhow::ensure!(qty > 0, "qty must be positive");
```

### Printing Errors

```rust
println!("{}", err);      // top message only
println!("{:#}", err);    // top + chain
println!("{:?}", err);    // Debug — full chain + backtrace
```

For log output prefer `{:#}` or `Debug`; for user-facing output prefer `{}`.

### Migration Tip

Already using `Box<dyn Error>`? Migrating to `anyhow` is mostly:

```rust
- fn foo() -> Result<(), Box<dyn Error>>
+ fn foo() -> anyhow::Result<()>
```

Conversions via `?` already work — `anyhow::Error` implements `From<E: Error>`.

> [!NOTE]
> The error story is one of Rust's biggest UX wins over older systems languages. Use it: define typed errors at library boundaries, propagate freely with `?`, attach context as you go, and let `main` print the chain.

### Interview Follow-ups

- *"How do you propagate errors across async tasks?"* — Same as sync, plus `JoinError` from the runtime. Either map it inside the task or surface via `JoinSet::join_next`.
- *"What's `eyre` / `color-eyre`?"* — `eyre` is a fork of `anyhow` with pluggable reporters; `color-eyre` formats nicely for terminals. Same shape.
- *"How does `?` work?"* — Calls `Try::branch` (stable: `From` on `Err` variant). Lets you propagate as long as the error types implement `From`.
