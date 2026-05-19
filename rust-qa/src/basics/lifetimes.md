# Q: What are lifetimes in Rust, and when do you need to annotate them?

**Answer:**

A **lifetime** is a compile-time label that tells the borrow checker *how long a reference is valid*. They are not runtime data. They exist so the compiler can prove every reference outlives whatever it points at.

### The Core Rule

A reference must never outlive the data it borrows.

```rust
fn dangling() -> &String {       // ERROR: missing lifetime
    let s = String::from("hi");
    &s                            // s dropped at end of fn — reference would dangle
}
```

The compiler doesn't trust you to track this; it requires every reference's lifetime to be derivable.

### Annotating Lifetimes

```rust
fn longest<'a>(x: &'a str, y: &'a str) -> &'a str {
    if x.len() > y.len() { x } else { y }
}
```

`'a` is a generic lifetime parameter — like a generic type but for "how long this reference is alive." The signature says: the returned reference lives at least as long as the *shorter* of the two inputs.

You're not picking a specific duration. You're stating a *relationship*: "return outlives `'a`, and both inputs outlive `'a`."

### Lifetime Elision Rules

The compiler infers lifetimes in common cases — you only annotate when ambiguous. The three rules:

1. Each input reference gets its own lifetime: `fn f(x: &T, y: &U)` → `fn f<'a,'b>(x: &'a T, y: &'b U)`.
2. If there's exactly one input lifetime, it's assigned to all output references.
3. If `&self` or `&mut self` is present, its lifetime is assigned to all output references.

```rust
fn first(s: &str) -> &str { &s[..1] }              // OK (rule 2)
fn pick<'a>(a: &'a str, b: &str) -> &'a str { a }  // must annotate; ambiguous (rule 1 only)
impl Foo { fn name(&self) -> &str { &self.name } } // OK (rule 3)
```

### Lifetimes in Structs

A struct that holds a reference must declare its lifetime:

```rust
struct Parser<'src> {
    input: &'src str,
    pos: usize,
}

impl<'src> Parser<'src> {
    fn peek(&self) -> Option<&'src char> { ... }
}
```

The struct cannot outlive the string it points into. Same rule, just made explicit at the type level.

### `'static`

```rust
let s: &'static str = "hello";   // string literal, lives forever
```

`'static` means "for the entire program." Useful for:
- String literals in the binary's read-only data.
- Global constants.
- Trait objects `Box<dyn Error + 'static>` (the default for `Send + 'static`).

Common mistake: `T: 'static` does **not** mean "the value lives forever." It means "the type contains no non-`'static` references" — i.e., it can be held arbitrarily long. An owned `String` is `'static` because it borrows nothing.

### Lifetime Subtyping & Variance

`'static: 'a` for any `'a` — `'static` is a subtype of every other lifetime. So `&'static str` can be passed where `&'a str` is expected.

Most references are **covariant** in their lifetime: `&'long T` can be used where `&'short T` is expected. `&mut T` is **invariant** — you can't shrink the lifetime, because it would let you write a short-lived reference into a long-lived slot.

### Worked Example: Why Two Annotations Differ

```rust
struct Buffer<'a> {
    data: &'a [u8],
}

impl<'a> Buffer<'a> {
    // ALL valid for `&'a`
    fn full(&self) -> &'a [u8] { self.data }

    // Returns a slice tied to &self's lifetime, NOT 'a
    fn head(&self) -> &[u8] { &self.data[..4] }
}
```

`full` returns a slice as long-lived as the original data (`'a`). `head` is tied to the temporary `&self` reference. They behave differently for callers:

```rust
let b = Buffer { data: &owned };
let h = b.head();
drop(b);                  // ERROR: h still borrowed
```

vs.

```rust
let h = b.full();
drop(b);                  // OK: h lives as long as `owned`, not b
```

### NLL (Non-Lexical Lifetimes)

Modern Rust shrinks borrows to *actual usage*, not lexical scope:

```rust
let mut v = vec![1, 2, 3];
let r = &v[0];            // immutable borrow starts
println!("{}", r);        // last use of r — borrow ends here
v.push(4);                // OK in NLL; would have been an error pre-2018
```

This eliminates most "obviously fine" errors that older Rust rejected.

### Common Errors & Fixes

| Error | Cause | Fix |
|-------|-------|-----|
| `borrowed value does not live long enough` | Returning a reference to a local | Return owned (`String`), or accept input by reference |
| `cannot infer an appropriate lifetime` | Two inputs, must say which the output ties to | Annotate explicitly |
| `requires that 'a outlive 'static` | You're putting `T: 'static` somewhere (often a `Box<dyn Trait>`) | Either use `'static` data or relax the bound |
| `cannot borrow as mutable...also borrowed as immutable` | Overlapping borrows | Shrink scope; restructure to use split borrows |

### When to Annotate

You **must** annotate when:
- A function returns a reference and elision rules don't disambiguate.
- A struct stores a reference.
- A trait method takes/returns multiple references and the relationship matters.

You **shouldn't** annotate when:
- Elision handles it (most simple cases).
- The signature reads more naturally with elision (avoid noise).

> [!NOTE]
> Lifetimes don't make programs slower — they're erased at compile time. They make compilation harder (mostly for the writer); they make runtime memory safety free.

### Interview Follow-ups

- *"Why doesn't Java need this?"* — Garbage collection. The price you pay is runtime overhead and non-deterministic destruction.
- *"What's higher-ranked trait bound (`for<'a>`)?"* — A bound saying "this works for *any* lifetime," needed for closures that accept references with caller-chosen lifetime.
- *"What's the difference between `&'a T` and `T: 'a`?"* — First is a reference with lifetime `'a`. Second is the bound "all references inside T live at least as long as `'a`."
