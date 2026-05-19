# Q: How do you idiomatically work with `Option` and `Result` combinators?

**Answer:**

Pattern-matching every `Option`/`Result` quickly becomes noisy. Rust provides a rich set of combinators that let you express transformations as a pipeline. Knowing them is one of the biggest readability wins in real Rust code.

### `Option` essentials

| Combinator         | Behavior                                                     |
|--------------------|--------------------------------------------------------------|
| `map(f)`           | `Some(x) -> Some(f(x))`, `None -> None`                      |
| `and_then(f)`      | flatMap: `f` returns `Option<U>`                             |
| `or(other)`        | falls back to `other` (eagerly evaluated)                    |
| `or_else(f)`       | falls back, lazily                                           |
| `unwrap_or(d)`     | unwrap or default                                            |
| `unwrap_or_else(f)`| unwrap or lazily computed default                            |
| `filter(p)`        | `Some(x)` if `p(&x)` else `None`                             |
| `ok_or(e)`         | `Some(x) -> Ok(x)`, `None -> Err(e)`                         |
| `take()`           | replaces with `None`, returns the previous value             |
| `replace(v)`       | replaces with `Some(v)`, returns the previous value          |
| `as_ref()`         | `&Option<T> -> Option<&T>`                                   |

```rust
let user: Option<String> = Some("alice".into());
let greeting: Option<String> = user.as_ref().map(|n| format!("hi, {n}"));
```

### `Result` essentials

| Combinator         | Behavior                                                     |
|--------------------|--------------------------------------------------------------|
| `map(f)`           | maps `Ok`                                                    |
| `map_err(f)`       | maps `Err`                                                   |
| `and_then(f)`      | chain another `Result`-returning step                        |
| `ok()`             | `Result<T,E> -> Option<T>`                                   |
| `err()`            | `Result<T,E> -> Option<E>`                                   |
| `unwrap_or_default()`| `T::default()` on `Err`                                    |

### The `?` operator

The single most common combinator alternative:

```rust
fn read_int(path: &str) -> Result<i32, Box<dyn std::error::Error>> {
    let s = std::fs::read_to_string(path)?;
    let n: i32 = s.trim().parse()?;
    Ok(n)
}
```

`?` calls `From::from` on the error so different error types compose. With libraries like `anyhow` or `thiserror`, error conversion is essentially free.

### `let ... else` for early return

```rust
let Some(user) = lookup(id) else {
    return Err("not found".into());
};
```

### Collecting into `Result<Vec<_>, _>`

```rust
let nums: Result<Vec<i32>, _> = ["1","2","3"].iter().map(|s| s.parse()).collect();
```

Short-circuits at the first `Err`.

### Anti-patterns

- `.unwrap()` in production code without a clear panic reason. Prefer `.expect("why this can't fail")` at minimum, or proper propagation.
- Chains of `match opt { Some(x) => match foo(x) { ... } }` — flatten with `and_then`.
- `if let Some(x) = opt { x } else { return default; }` — use `unwrap_or` / `unwrap_or_else`.

### Visual: Option pipeline

```text
Some(s) --map(parse)--> Some(Ok(n)) --transpose--> Ok(Some(n))
                                                       |
                                                       v
                                                  use with `?`
```

> [!NOTE]
> The fluent combinator style usually reads better than nested `match`. Memorize `map`, `and_then`, `unwrap_or_else`, `ok_or`, and `?`; they cover 90% of real-world Option/Result code.
