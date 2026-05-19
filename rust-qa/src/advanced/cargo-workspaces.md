# Q: How do Cargo workspaces, features, and dependency resolution work?

**Answer:**

Cargo is more than a build tool — it's a **dependency manager, build orchestrator, and test runner**. Three concepts unlock 90% of multi-crate projects: workspaces, features, and the resolver.

### Workspaces

A workspace is multiple related crates sharing one `Cargo.lock`, `target/`, and dependency resolution pass.

```
my-app/
├── Cargo.toml          <- workspace manifest
├── Cargo.lock
├── target/
├── api/
│   ├── Cargo.toml
│   └── src/lib.rs
├── worker/
│   ├── Cargo.toml
│   └── src/main.rs
└── shared/
    ├── Cargo.toml
    └── src/lib.rs
```

Root `Cargo.toml`:

```toml
[workspace]
resolver = "2"
members  = ["api", "worker", "shared"]

[workspace.dependencies]
serde   = { version = "1", features = ["derive"] }
tokio   = { version = "1", features = ["full"] }
shared  = { path = "shared" }

[workspace.package]
edition = "2021"
license = "MIT"
```

Member `Cargo.toml`:

```toml
[package]
name    = "api"
edition.workspace = true
license.workspace = true

[dependencies]
serde.workspace  = true
shared.workspace = true
```

Benefits:
- One `cargo build` builds everything.
- One `Cargo.lock` = guaranteed identical dep versions across crates.
- Shared `target/` = no duplicated compiles.
- `cargo test --workspace` runs all member tests.

### Features

Features are **compile-time flags** that enable optional code and dependencies.

```toml
[features]
default = ["json"]
json    = ["dep:serde_json"]
mysql   = ["dep:sqlx", "sqlx/mysql"]
postgres = ["dep:sqlx", "sqlx/postgres"]

[dependencies]
serde_json = { version = "1", optional = true }
sqlx       = { version = "0.7",  optional = true }
```

Then in code:

```rust
#[cfg(feature = "json")]
pub mod json;

pub fn parse(input: &str) -> Value {
    #[cfg(feature = "json")]
    return serde_json::from_str(input).unwrap();
    #[cfg(not(feature = "json"))]
    todo!()
}
```

Enable from CLI:

```bash
cargo build --no-default-features --features postgres
```

### Feature Unification (The Big Footgun)

Cargo **unifies features across a build graph**. If crate A enables `serde/derive` and crate B enables `serde/rc`, Cargo builds serde with `derive + rc`.

Implication: a feature you enable in dev-dependencies can leak into production builds if the same dependency is used both places.

Mitigation: resolver v2 (workspace setting `resolver = "2"`) treats `dev-dependencies`, `build-dependencies`, and target-specific deps as **separate** feature graphs.

### The Resolver

Cargo picks the **highest semver-compatible** version for each dependency. Conflicts within compatibility resolve to one version; cross-major-version requirements result in two compiled copies of the same crate (e.g., `serde 1.0.x` and `serde 2.0.x` coexisting).

```bash
cargo tree                  # full dep tree
cargo tree -d               # only duplicated crates
cargo tree -e features      # show enabled features
```

`Cargo.lock`:
- Library crates: don't commit (lets downstream resolve).
- Binary crates / workspaces with binaries: commit (reproducible builds).

### Dependency Sources

```toml
[dependencies]
local        = { path = "../local" }
git-dep      = { git = "https://github.com/...", tag = "v1.2.0" }
registry-dep = "1.5"                                # crates.io
private      = { registry = "my-corp" }             # alternate registry
```

`[patch.crates-io]` replaces a transitive dep across the whole graph — useful for hotfixes:

```toml
[patch.crates-io]
some-crate = { git = "https://github.com/me/fork", branch = "fix" }
```

### Profiles

Tune compilation per build mode:

```toml
[profile.dev]
opt-level = 0
debug = true

[profile.release]
opt-level = 3
lto       = "fat"          # link-time optimization
codegen-units = 1
strip     = true
panic     = "abort"

[profile.release-with-debug]
inherits = "release"
debug    = true
```

```bash
cargo build --release
cargo build --profile release-with-debug
```

### Useful Cargo Subcommands

| Command | Purpose |
|---------|---------|
| `cargo check` | Type-check without codegen — fast feedback |
| `cargo clippy -- -D warnings` | Linter, deny warnings |
| `cargo fmt` | rustfmt |
| `cargo test --workspace --all-features` | Test everything |
| `cargo nextest run` | Faster test runner (third-party) |
| `cargo deny check` | Audit licenses, advisories, banned crates |
| `cargo udeps` | Find unused dependencies |
| `cargo bloat --release` | What's making the binary big |
| `cargo machete` | Even simpler unused-dep finder |

### Build Scripts

`build.rs` runs before compile:

```rust
fn main() {
    println!("cargo:rustc-env=GIT_HASH={}", git_hash());
    println!("cargo:rerun-if-changed=schema.sql");
}
```

Common uses: codegen from `.proto`/`.sql`, compile-time constants, native lib linking.

### Common Mistakes

| Mistake | Fix |
|---------|-----|
| Forgetting `resolver = "2"` in workspace | Use it — v1 has the dev-dep feature leak |
| Committing `Cargo.lock` in a library | Don't; downstream needs version flexibility |
| Renaming features without `--features` plumbing | All downstream consumers break silently |
| Heavy crate enabled via `default` features | Make heavy parts opt-in |
| Duplicate versions of a crate | `cargo tree -d`, use `[patch]` or update transitive deps |

### Cross Compilation

```bash
rustup target add x86_64-unknown-linux-musl
cargo build --release --target x86_64-unknown-linux-musl
```

For glibc/Linux distroless deploys, `musl` static binaries are common. Use `cross` (containerized) for non-trivial targets.

> [!NOTE]
> Cargo's most underrated feature is workspaces. Even a "small" project benefits from splitting into a `lib` + `bin` crate — keeps test compilation fast and forces you to design library boundaries.

### Interview Follow-ups

- *"What's the difference between `rustup` and `cargo`?"* — `rustup` manages toolchains (compiler versions, components, targets). `cargo` is the project tool that uses one of those toolchains.
- *"Why are LTO and codegen-units=1 slow?"* — LTO inlines across crates; cu=1 disables parallel codegen. Together: best perf, worst build time. Use for releases only.
- *"Can you use git deps in published crates?"* — No — crates.io requires all transitive deps from crates.io.
