# Q: How do you test Rust code — unit, integration, doctests, property tests?

**Answer:**

Cargo treats tests as first-class. There are **five** distinct kinds, each with its own location and runner.

### 1. Unit Tests (Inside the Module)

```rust
// src/math.rs
pub fn add(a: i32, b: i32) -> i32 { a + b }

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn adds_positive() {
        assert_eq!(add(2, 3), 5);
    }

    #[test]
    #[should_panic(expected = "overflow")]
    fn panics_on_overflow() {
        add(i32::MAX, 1);
    }
}
```

- `#[cfg(test)]` means the module is compiled only for `cargo test`.
- Can test **private items** because they're in the same module.
- Run with `cargo test`.

### 2. Integration Tests (Black-Box, Outside the Crate)

```
my-crate/
├── src/lib.rs
└── tests/
    ├── http_api.rs
    └── db.rs
```

```rust
// tests/http_api.rs
use my_crate::Server;

#[tokio::test]
async fn handles_get() {
    let s = Server::start().await;
    let r = reqwest::get(s.url("/health")).await.unwrap();
    assert_eq!(r.status(), 200);
}
```

- Each `.rs` file under `tests/` is a separate binary.
- Only **public** API is reachable — these are consumer tests.
- Share helpers via `tests/common/mod.rs`.

### 3. Doctests (Examples in Documentation)

```rust
/// Adds two numbers.
///
/// # Examples
///
/// ```
/// use my_crate::math::add;
/// assert_eq!(add(2, 3), 5);
/// ```
pub fn add(a: i32, b: i32) -> i32 { a + b }
```

- Run as part of `cargo test`.
- Verify your docs stay accurate as code changes.
- For non-runnable examples: ```` ```no_run ```` or ```` ```ignore ````.
- For examples that should fail to compile: ```` ```compile_fail ````.

### 4. Benchmarks (Criterion)

```toml
[dev-dependencies]
criterion = "0.5"

[[bench]]
name = "math_bench"
harness = false
```

```rust
// benches/math_bench.rs
use criterion::{black_box, criterion_group, criterion_main, Criterion};

fn bench_add(c: &mut Criterion) {
    c.bench_function("add 2,3", |b| b.iter(|| add(black_box(2), black_box(3))));
}

criterion_group!(benches, bench_add);
criterion_main!(benches);
```

`cargo bench`. `black_box` prevents the optimizer from constant-folding the call away.

### 5. Property-Based Tests (proptest / quickcheck)

Generate random inputs and check invariants.

```rust
use proptest::prelude::*;

proptest! {
    #[test]
    fn add_commutative(a in any::<i32>(), b in any::<i32>()) {
        prop_assume!(a.checked_add(b).is_some());
        assert_eq!(a.wrapping_add(b), b.wrapping_add(a));
    }
}
```

Proptest **shrinks** failing inputs to the minimal counterexample — much better than `quickcheck` for diagnosis.

### Cargo Test Mechanics

```bash
cargo test                          # run all tests
cargo test --lib                    # unit tests only
cargo test --test http_api          # one integration file
cargo test add_                     # filter by name substring
cargo test -- --nocapture           # show println! output
cargo test -- --test-threads=1      # serialize tests
cargo test --release                # optimized build (for slow tests)
```

`cargo nextest run` (third-party) is significantly faster — process-per-test isolation, retries, JUnit XML output.

### Setup / Teardown

No JUnit `@Before` — use RAII guards or helper functions:

```rust
struct TestCtx { db: TempDir, _server: ServerHandle }

impl TestCtx {
    fn new() -> Self { ... }
}

impl Drop for TestCtx {
    fn drop(&mut self) { /* cleanup happens automatically */ }
}

#[test]
fn it_works() {
    let ctx = TestCtx::new();
    // ...
    // ctx dropped → cleanup at end
}
```

For one-time setup: `std::sync::OnceLock` or the `ctor` crate (`#[ctor]`).

### Async Tests

```rust
#[tokio::test]
async fn it_works() {
    fetch().await;
}

#[tokio::test(flavor = "multi_thread", worker_threads = 4)]
async fn parallel() { ... }
```

For other runtimes: `#[async_std::test]`, `#[smol_potat::test]`.

### Mocking and Test Doubles

Rust doesn't have reflection — mocks are explicit.

```rust
// trait, impl in lib for prod, impl in tests for fake
pub trait Clock {
    fn now(&self) -> Instant;
}

pub struct SystemClock;
impl Clock for SystemClock {
    fn now(&self) -> Instant { Instant::now() }
}

#[cfg(test)]
struct FakeClock { t: Instant }
#[cfg(test)]
impl Clock for FakeClock {
    fn now(&self) -> Instant { self.t }
}
```

Or use crates: **`mockall`** (most popular), **`faux`**, **`mockito`** for HTTP.

### Test Containers

For real-dependency integration tests:

```rust
use testcontainers::{clients::Cli, images::postgres::Postgres};

#[tokio::test]
async fn db_roundtrip() {
    let docker = Cli::default();
    let pg = docker.run(Postgres::default());
    let conn_str = format!("postgres://postgres@localhost:{}/postgres",
                            pg.get_host_port_ipv4(5432));
    // ... use conn_str
}
```

Spins up real Postgres per test (or shared via `OnceLock` if you trust isolation).

### Code Coverage

```bash
cargo install cargo-tarpaulin
cargo tarpaulin --out html --output-dir coverage
```

Or `cargo llvm-cov` — faster, uses LLVM's source-based coverage.

### Fuzz Testing

```bash
cargo install cargo-fuzz
cargo fuzz init
# Writes fuzz/fuzz_targets/fuzz_target_1.rs
cargo fuzz run fuzz_target_1
```

Uses libFuzzer / AFL via the nightly compiler. Crashes are saved as inputs you can reproduce.

### Snapshot Tests

```rust
use insta::assert_yaml_snapshot;

#[test]
fn renders() {
    let out = render(&data);
    assert_yaml_snapshot!(out);
}
```

First run: stores `.snap` file. Subsequent runs: diff against it. `cargo insta review` for interactive updates. Great for parser/AST tests.

### Conventions

- **Unit tests** in same file: small, fast, test private logic.
- **Integration tests** under `tests/`: exercise public API, real I/O if needed.
- **Doctests**: as live examples — readers see they work.
- **Properties / fuzzing**: invariants under random input.

### Test Output and CI

```bash
cargo test --no-fail-fast               # don't stop on first failure
cargo test 2>&1 | tee test.log
```

JUnit XML (for CI): use `cargo nextest run --message-format=junit > junit.xml`.

### Common Mistakes

| Mistake | Fix |
|---------|-----|
| Big setup in every test | Use helper functions or RAII context |
| Tests sharing global state (env vars, current_dir) | `--test-threads=1` or use named env scopes |
| Forgetting `#[tokio::test]` for async | Compiler error: "async fn outside trait" / unused future |
| Doctests that compile but don't run | They run by default; use `no_run` only when needed |
| Slow tests in unit layer | Move to integration or mark `#[ignore]` and run separately |

> [!NOTE]
> Cargo's test ergonomics are a feature. The friction of "writing a test" should be near-zero — small file edits, no XML, no harness. Use it.

### Interview Follow-ups

- *"How do you test code that calls `std::process::exit`?"* — Refactor to return a value; test the value. Or shell out and check exit code.
- *"Difference between `unit` and `integration` tests in Rust vs Java?"* — Rust's distinction is purely about visibility/location, not technique. JUnit's idea of "unit" is closer to Rust's `#[test]` inside `mod tests`.
- *"How do you parallelize tests safely with shared resources?"* — Use `serial_test` crate, or design tests to share via `OnceLock` + locks.
