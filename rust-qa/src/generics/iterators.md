# Q: How do Rust iterators work, and why are they zero-cost?

**Answer:**

An iterator in Rust is any type implementing the `Iterator` trait:

```rust
pub trait Iterator {
    type Item;
    fn next(&mut self) -> Option<Self::Item>;
    // dozens of default methods: map, filter, fold, collect, ...
}
```

Each call to `next` produces `Some(item)` or `None` when exhausted. All combinators (`map`, `filter`, `take`, ...) are built on top of `next`.

### Lazy by design

Adapters do nothing until consumed. The following allocates nothing and does no work:

```rust
let it = (1..).map(|x| x * 2).filter(|x| x % 3 == 0); // nothing happens
```

Consumers like `collect`, `sum`, `for`, `fold` drive the chain.

### Why they are zero-cost

Each adapter is a small struct. Chained adapters compose into one big nested struct, and `next` on the outermost struct inlines all inner `next` calls. After optimization, the generated assembly is typically identical to a hand-written loop.

```text
v.iter().map(f).filter(g).sum::<i32>()

inlines to roughly:

let mut acc = 0;
for &x in &v {
    let y = f(x);
    if g(&y) { acc += y; }
}
```

### Three iteration flavors

| Method        | Yields          | Borrowing       |
|---------------|-----------------|-----------------|
| `iter()`      | `&T`            | shared borrow   |
| `iter_mut()`  | `&mut T`        | exclusive borrow|
| `into_iter()` | `T`             | takes ownership |

`for x in &v` desugars to `for x in v.iter()`; `for x in v` desugars to `for x in v.into_iter()`.

### `collect` and turbofish

```rust
let v: Vec<i32> = (0..5).collect();
let v = (0..5).collect::<Vec<i32>>();   // turbofish
```

`collect` is generic over any `FromIterator` target: `Vec`, `HashMap`, `String`, `Result<Vec<_>, _>`, etc. The last is especially useful:

```rust
let parsed: Result<Vec<i32>, _> = ["1", "2", "bad"].iter().map(|s| s.parse()).collect();
```

`collect` short-circuits on the first `Err`.

### Custom iterators

Implementing one is straightforward:

```rust
struct Counter(u32);
impl Iterator for Counter {
    type Item = u32;
    fn next(&mut self) -> Option<u32> {
        self.0 += 1;
        if self.0 <= 5 { Some(self.0) } else { None }
    }
}
```

You instantly get `.map`, `.filter`, `.sum`, etc.

### Useful adapters

- `enumerate`, `zip`, `chain`, `take_while`, `skip_while`, `peekable`, `windows` (on slices), `chunks`, `scan`, `flat_map`.

> [!NOTE]
> Reach for iterators before manual loops. They are typically just as fast, far more compositional, and the compiler catches off-by-one errors that manual indexing introduces.
