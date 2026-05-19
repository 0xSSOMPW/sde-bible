# Q: What is the difference between declarative macros (`macro_rules!`) and procedural macros?

**Answer:**

Rust has two macro systems. Both run at compile time and emit token streams, but they differ in power, complexity, and where they live.

### Declarative macros (`macro_rules!`)

Pattern-matching over token trees. Lives in the same crate; no extra build setup.

```rust
macro_rules! square {
    ($x:expr) => { { let v = $x; v * v } };
}

let n = square!(3 + 1); // expands to a block
```

Common fragment specifiers:

| Specifier | Matches                       |
|-----------|-------------------------------|
| `expr`    | an expression                 |
| `ident`   | an identifier                 |
| `ty`      | a type                        |
| `pat`     | a pattern                     |
| `stmt`    | a statement                   |
| `block`   | a `{ ... }` block             |
| `tt`      | any single token tree         |
| `path`    | a path                        |
| `literal` | a literal                     |

Repetition uses `$( ... ),*` / `$( ... );*` / `$( ... )?` etc.

```rust
macro_rules! vec_of {
    ($($x:expr),* $(,)?) => {{
        let mut v = Vec::new();
        $( v.push($x); )*
        v
    }};
}
```

### Procedural macros

Compile-time Rust functions that take a `TokenStream` and return a `TokenStream`. Must live in their own crate of type `proc-macro = true`.

Three flavors:

1. **Function-like**: `my_macro!(...)`.
2. **Derive**: `#[derive(MyTrait)]` on a struct/enum.
3. **Attribute**: `#[my_attr] fn foo() {}` or on items.

Typical skeleton:

```rust
use proc_macro::TokenStream;

#[proc_macro_derive(Hello)]
pub fn derive_hello(input: TokenStream) -> TokenStream {
    let ast: syn::DeriveInput = syn::parse(input).unwrap();
    let name = &ast.ident;
    quote::quote! {
        impl Hello for #name {
            fn hello() { println!("hi from {}", stringify!(#name)); }
        }
    }
    .into()
}
```

Real-world examples: `serde::Serialize`, `tokio::main`, `thiserror::Error`.

### Trade-offs

| Aspect                       | `macro_rules!`        | proc macros                 |
|------------------------------|-----------------------|-----------------------------|
| Build cost                   | None                  | Extra crate, longer build   |
| Power                        | Pattern matching only | Full Rust at compile time   |
| Debuggability                | `cargo expand`        | `cargo expand` + log output |
| Hygiene                      | Mostly hygienic       | Manual; `Span` matters      |
| IDE support                  | Good                  | Improving                   |

### When to use which

- Use `macro_rules!` for **syntactic sugar**: builders, DSLs, repeated boilerplate that can be expressed by pattern matching.
- Use proc macros when you need to **inspect types or generics** (e.g., generate trait impls from struct fields) or implement **custom derives/attributes**.

### Pitfalls

- Macros that re-parse user expressions repeatedly can blow up compile times.
- Error spans in macros can be confusing; use `#[track_caller]` and proc-macro `Span` carefully.
- Overusing macros hurts readability. Prefer regular functions when generics suffice.

> [!NOTE]
> Reach for macros only after generics, traits, and plain functions cannot express the abstraction. They are powerful but they are also the easiest way to make a codebase impenetrable.
