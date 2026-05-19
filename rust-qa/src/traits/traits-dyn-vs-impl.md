# Q: `dyn Trait` vs `impl Trait` — static vs dynamic dispatch in Rust.

**Answer:**

Both let a function work with "some type implementing this trait," but they're fundamentally different mechanisms with different cost, flexibility, and limitations.

### `impl Trait` — Static Dispatch (Monomorphization)

```rust
fn make_greeter() -> impl Fn() -> String {
    || String::from("hi")
}

fn print_lines<W: Write>(w: &mut W, lines: &[&str]) -> io::Result<()> {
    for l in lines { writeln!(w, "{}", l)?; }
    Ok(())
}
```

At compile time, the compiler generates a **specialized copy** for every concrete type used at a call site. The call is a direct, inlinable function call.

- Zero indirection. Often inlined.
- Aggressive optimization (the optimizer sees the concrete type).
- Binary grows with each instantiation (code bloat).

### `dyn Trait` — Dynamic Dispatch (vtable)

```rust
fn print_lines(w: &mut dyn Write, lines: &[&str]) -> io::Result<()> {
    for l in lines { writeln!(w, "{}", l)?; }
    Ok(())
}

let writers: Vec<Box<dyn Write>> = vec![
    Box::new(File::create("a")?),
    Box::new(io::stdout()),
];
```

The compiler builds one function that takes a **fat pointer** = (data pointer, vtable pointer). Each method call indirects through the vtable.

- Single function in the binary (no monomorphization).
- One pointer indirection per virtual call (~no measurable cost in non-tight loops; non-inlinable).
- Lets you store **heterogeneous types** in one collection.

### The Fat Pointer

```
&dyn Write  =  | data ptr | vtable ptr |
                    │           │
                    ▼           ▼
              actual struct   [drop, size, align, write, flush, ...]
```

`&dyn Trait` is two words. `&T` is one word.

### When to Use Which

| Need | Use |
|------|-----|
| Pure performance, one type at a call site | `impl Trait` / generics |
| Heterogeneous collection | `Vec<Box<dyn Trait>>` |
| Plugin systems, return type chosen at runtime | `Box<dyn Trait>` |
| Trait objects through a function boundary | `&dyn Trait` |
| Reducing binary size in a large codebase | `dyn` (one body vs many) |
| Async trait methods | `Box<dyn Future>` (or `async-trait`) |

### Where `impl Trait` Can Appear

- **Argument position**: `fn f(x: impl Trait)` — equivalent to `fn f<T: Trait>(x: T)`.
- **Return position**: `fn make() -> impl Trait` — caller can't name the type; useful for closures, iterators.
- **Type-position in lets** (since 1.26): not allowed directly; you write `let x: Box<dyn Trait> = ...` or use TAIT (`type Alias = impl Trait;`).

### Object Safety

Not every trait can be used with `dyn`. The trait must be **object-safe**:

- No `Self` in method signatures (other than `&self`/`&mut self`).
- No generic methods.
- No associated constants (mostly).

```rust
trait Bad {
    fn dup(&self) -> Self;          // ❌ returns Self by value
    fn parse<T>(s: &str) -> T;      // ❌ generic method
}
// Bad is not object-safe; `dyn Bad` won't compile.
```

If you need both styles, split:

```rust
trait Reader {
    fn read(&mut self, buf: &mut [u8]) -> usize;          // object-safe
}

trait ReaderExt: Reader {
    fn read_to_string(&mut self) -> String { ... }        // non-object-safe extension via default
}
```

### Mixing Them

You can take `&mut dyn Read` from inside a generic function:

```rust
fn read_all<R: Read>(mut r: R) {
    fn inner(r: &mut dyn Read) { /* one body */ }
    inner(&mut r);
}
```

Common pattern to keep the generic API but only one copy of the heavy body in the binary.

### Closures: Same Choice

```rust
fn map_static<F: Fn(i32) -> i32>(v: Vec<i32>, f: F) -> Vec<i32> { ... }   // static
fn map_dyn(v: Vec<i32>, f: Box<dyn Fn(i32) -> i32>) -> Vec<i32> { ... }   // dynamic
```

`Box<dyn Fn>` is what you need to store a closure in a struct field of fixed type:

```rust
struct Handler {
    on_event: Box<dyn Fn(Event) + Send>,
}
```

### Performance Reality

For a CPU-bound tight loop calling a tiny method:
- `impl Trait` lets the compiler inline → can be 5–10× faster.

For business logic, network code, allocation-heavy work:
- The vtable lookup is dwarfed by the surrounding work; difference is unmeasurable.

Profile before optimizing.

### Common Mistakes

| Mistake | Fix |
|---------|-----|
| `Vec<impl Trait>` (compile error: each element must be same type) | `Vec<Box<dyn Trait>>` for heterogeneity |
| Forgetting `+ 'static` on `Box<dyn Trait>` when needed | The default is `+ 'static`; for shorter lifetimes write `Box<dyn Trait + 'a>` |
| Storing `dyn Future` directly (it's `!Sized`) | Use `Pin<Box<dyn Future<Output=T>>>` |
| Using `dyn` everywhere "for flexibility" | Pay code-size cost without benefit — generics are cheap to add |

> [!NOTE]
> `impl Trait` is the right default. Reach for `dyn` when you genuinely need erasure — heterogeneous storage, plugin boundaries, FFI shims, or to bound binary size.

### Interview Follow-ups

- *"What does `Box<dyn Error>` give you?"* — A single error type that hides specific source types. Useful for `main` and at API boundaries.
- *"`&dyn Trait` vs `Box<dyn Trait>`?"* — Reference doesn't own; Box does. Same vtable mechanics.
- *"Why `+ Send` on `dyn`?"* — Defaults to `+ 'static` but not `+ Send`. Add it when sharing across threads.
