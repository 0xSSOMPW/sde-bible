# Q: How do `#[derive]` macros and blanket impls actually work?

**Answer:**

Two of the most heavily used trait features in Rust — `#[derive(Clone)]` and impls like `impl<T: Display> ToString for T` — look like compiler magic. They're both regular Rust mechanisms once you know what they do.

### `#[derive(...)]`

`#[derive]` is a **procedural macro** that expands at compile time, generating a trait impl for your type.

```rust
#[derive(Debug, Clone, PartialEq)]
struct Point { x: i32, y: i32 }
```

After macro expansion:

```rust
struct Point { x: i32, y: i32 }

impl std::fmt::Debug for Point {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        f.debug_struct("Point").field("x", &self.x).field("y", &self.y).finish()
    }
}

impl Clone for Point {
    fn clone(&self) -> Self { Self { x: self.x.clone(), y: self.y.clone() } }
}

impl PartialEq for Point {
    fn eq(&self, other: &Self) -> bool { self.x == other.x && self.y == other.y }
}
```

You can inspect the expansion with `cargo expand` (third-party command).

### The Standard Derives

| Trait | Generates |
|-------|-----------|
| `Debug` | Pretty-print with field names |
| `Clone` | Deep copy by `.clone()`-ing each field |
| `Copy` | Marker — requires all fields `Copy` |
| `PartialEq` / `Eq` | Field-by-field equality |
| `PartialOrd` / `Ord` | Lexicographic by field order |
| `Hash` | Hash each field in order |
| `Default` | Zero/empty value per field |

### Constraints on Auto-Derive

A derive generates an impl whose bounds are **the same trait on every field type**:

```rust
#[derive(Clone)]
struct Wrap<T> { inner: T }
// generated: impl<T: Clone> Clone for Wrap<T>
```

So `Wrap<String>` is `Clone`, but `Wrap<File>` is not (because `File: !Clone`). The bound is auto-added.

This causes occasional surprises with `PhantomData`:

```rust
struct Holder<T> { data: i32, _t: PhantomData<T> }

// derive(Clone) generates: impl<T: Clone> Clone for Holder<T>
// even though T is never stored — the bound is wrong but conservative
```

Fix: implement manually or use `derivative` crate for finer control.

### Custom Derive Macros

Many crates ship derive macros: `thiserror`, `serde`, `clap`, `tracing`, `bevy_ecs`. They expand to typed impls for their own traits.

```rust
#[derive(serde::Serialize, serde::Deserialize)]
struct Order { id: u64, items: Vec<String> }
```

Generated: `impl Serialize for Order { ... }` walking each field.

Build your own:

```rust
// my_derive/src/lib.rs
use proc_macro::TokenStream;

#[proc_macro_derive(Hello)]
pub fn derive_hello(input: TokenStream) -> TokenStream {
    let ast: syn::DeriveInput = syn::parse(input).unwrap();
    let name = &ast.ident;
    quote::quote! {
        impl #name { pub fn hello() { println!("hi from {}", stringify!(#name)); } }
    }.into()
}
```

Procedural macros live in a separate crate with `proc-macro = true`.

### Blanket Impls

A blanket impl applies to **every** type matching a bound:

```rust
impl<T: Display> ToString for T {
    fn to_string(&self) -> String { format!("{}", self) }
}
```

That single impl makes `42.to_string()` work for `i32`, `&str`, your custom types — anything implementing `Display`.

`From`/`Into` is the textbook example:

```rust
impl<T, U> Into<U> for T where U: From<T> {
    fn into(self) -> U { U::from(self) }
}
```

You implement `From`, you get `Into` for free.

### The Orphan Rule

You can implement a trait for a type only if **either** the trait **or** the type is defined in your crate:

```rust
// In your crate:
impl Display for MyType { ... }      // ✅ your type
impl MyTrait for String { ... }      // ✅ your trait

impl Display for String { ... }      // ❌ both foreign
```

This prevents two crates from implementing the same trait for the same type. Workaround: the **newtype pattern**.

```rust
struct MyString(String);
impl Display for MyString { ... }   // ✅
```

### Trait Coherence

A consequence of the orphan rule: only **one** blanket impl can exist for any type/trait combination. If the standard library has `impl<T: Display> ToString for T`, you cannot also `impl<T: Custom> ToString for T` — conflicts everywhere they overlap.

This sometimes blocks "obviously fine" code. The compiler's error: `conflicting implementations`. Fix: trait-narrowed traits (`MyDisplay`) or newtypes.

### Marker Traits

Some traits have no methods — their *presence* is the contract:

```rust
unsafe trait Send {}
unsafe trait Sync {}

trait Copy: Clone {}     // also empty body
trait Eq: PartialEq {}   // empty body — promises reflexivity
```

The compiler reads these as flags. `Copy` implies the type is bit-copyable. `Send` says "movable across threads."

### Default Method Implementations

Traits can ship default bodies:

```rust
trait Greeter {
    fn name(&self) -> &str;
    fn greet(&self) -> String { format!("hi, {}", self.name()) }
}
```

Implementors can override `greet`. Often used to provide a method derivable from other trait methods.

### Associated Types vs Generics

```rust
// Generic — implementor picks T per impl
trait Container<T> {
    fn put(&mut self, x: T);
}

// Associated type — implementor has exactly one
trait Container {
    type Item;
    fn put(&mut self, x: Self::Item);
}
```

Use associated types when **one type makes sense per implementor** (`Iterator::Item`); use generics when you want **multiple impls** for the same type with different parameters.

### Super Traits

```rust
trait Serialize: Debug { ... }    // Serialize requires Debug
```

To impl `Serialize` for `T`, `T` must also impl `Debug`. Lets the trait's default methods rely on the supertrait.

### Common Mistakes

| Mistake | Fix |
|---------|-----|
| Deriving `Copy` for types containing `String` | Compile error — `String` isn't `Copy`. Use `Clone`. |
| Deriving `Default` for fieldless enums | Doesn't work — `Default` needs a value; manually impl |
| Trying to impl foreign trait on foreign type | Orphan rule; use newtype |
| `#[derive(Clone)]` on generic without proper bounds | The auto-bound `T: Clone` may be wrong; impl manually |
| Conflicting blanket impl with std | Use a narrower trait or newtype |

> [!NOTE]
> Reach for derives reflexively for data types. Only hand-write when a derive would generate something wrong (PhantomData, custom `Debug` for redaction, etc.).

### Interview Follow-ups

- *"What's the difference between `derive` and `proc_macro_attribute`?"* — `derive` only adds *new* impl items; attribute macros can rewrite the entire item.
- *"Can I derive a trait from a different crate?"* — Only if that crate exports a `#[proc_macro_derive]`. The trait itself doesn't need to be yours.
- *"How does `#[automatically_derived]` show up in errors?"* — Compiler marks derived impls so error messages attribute them to the derive, not to your code.
