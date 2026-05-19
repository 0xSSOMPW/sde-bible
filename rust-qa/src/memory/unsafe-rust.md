# Q: When is `unsafe` necessary in Rust, and what does it actually let you do?

**Answer:**

`unsafe` doesn't disable the borrow checker. It unlocks **five specific operations** the compiler can't statically verify. The programmer takes on the proof obligation. Everything outside those five remains checked.

### The Five Unsafe Superpowers

1. Dereference a raw pointer (`*const T`, `*mut T`).
2. Call an `unsafe fn` (including FFI).
3. Access or modify a `mut` `static`.
4. Implement an `unsafe trait` (`Send`, `Sync`, `GlobalAlloc`).
5. Access fields of a `union`.

Nothing else changes inside an `unsafe` block. `let x = 1 + 1;` is still checked.

### Why It Exists

Safe Rust forbids legitimate operations:
- FFI calls (the foreign function is opaque to the borrow checker).
- Low-level data structures (`Vec`, `HashMap` internals).
- Performance hacks (skip a bounds check the programmer just verified).
- Hardware access (memory-mapped registers, MMIO).

`unsafe` is the **escape hatch** with a contract: the writer guarantees the safety invariants the compiler can't.

### Anatomy of an `unsafe` Block

```rust
fn split_at_mut<T>(v: &mut [T], mid: usize) -> (&mut [T], &mut [T]) {
    let len = v.len();
    let ptr = v.as_mut_ptr();
    assert!(mid <= len);

    unsafe {
        (
            std::slice::from_raw_parts_mut(ptr, mid),
            std::slice::from_raw_parts_mut(ptr.add(mid), len - mid),
        )
    }
}
```

Borrow checker would reject this — two `&mut` to the same slice. The programmer proves they're disjoint (`mid <= len`, then non-overlapping ranges) and uses `unsafe` to construct them.

### The Soundness Boundary

A function is **sound** if no safe code can use it to cause undefined behavior. `split_at_mut` is sound: caller can't trigger UB no matter what `mid` and `v` they pass.

Sound `unsafe` is a building block. Unsound `unsafe` is a bug — even if it "works."

```rust
// UNSOUND — UB if i >= v.len()
unsafe fn get(v: &Vec<i32>, i: usize) -> i32 {
    *v.get_unchecked(i)
}

// Sound wrapper
fn get(v: &Vec<i32>, i: usize) -> i32 {
    assert!(i < v.len());
    unsafe { *v.get_unchecked(i) }
}
```

### Common UB to Avoid

| UB | Example | Result |
|----|---------|--------|
| Dereferencing dangling pointer | `&*Box::into_raw(b)` after `b` dropped | crash/garbage |
| Aliasing `&mut` | Two `&mut` to same data via raw pointers | LLVM miscompiles |
| Reading uninitialized memory | `MaybeUninit` without write | nasal demons |
| Breaking type invariants | `transmute::<u8, bool>(2)` | bool with value 2 — UB on use |
| Calling FFI with wrong ABI | `extern "C"` mismatch | crash |

### Raw Pointers vs References

```rust
let x = 5;
let r: &i32 = &x;             // reference: borrow-checked, non-null, aligned
let p: *const i32 = &x;       // raw pointer: nullable, no borrow tracking

unsafe { println!("{}", *p) } // dereferencing requires unsafe
```

Conversion `&T` → `*const T` is safe. The reverse and dereferencing are not.

### `MaybeUninit` for Uninitialized Memory

Reading uninitialized memory is UB. `MaybeUninit<T>` lets you allocate without initializing, then promise it's been filled:

```rust
use std::mem::MaybeUninit;

let mut buf: [MaybeUninit<u8>; 1024] = unsafe { MaybeUninit::uninit().assume_init() };
file.read_exact(unsafe { std::mem::transmute(&mut buf[..]) })?;
let initialized: &[u8] = unsafe { std::mem::transmute(&buf[..]) };
```

Avoids zero-initializing a buffer you're about to overwrite.

### FFI Pattern

```rust
extern "C" {
    fn strlen(s: *const i8) -> usize;
}

fn safe_strlen(s: &CStr) -> usize {
    unsafe { strlen(s.as_ptr()) }
}
```

Wrap the unsafe FFI call in a safe function with type-checked inputs. Inside, you uphold C's contract; outside, callers see only the safe API.

### `unsafe fn` vs `unsafe` block

```rust
unsafe fn dangerous() { ... }    // function's contract requires unsafe to call

fn safe_wrapper() {
    // safe to caller — wrapper upholds invariants
    unsafe { dangerous() }
}
```

`unsafe fn` propagates the obligation to callers. Use only when the function's correct use cannot be enforced internally.

### Tools to Help

| Tool | Catches |
|------|---------|
| `cargo miri` | Most UB at runtime — aliasing, OOB, uninit reads |
| `cargo +nightly fuzz` | Crash inputs |
| `cargo asan` (sanitizers) | Memory errors via LLVM ASan |
| `clippy::undocumented_unsafe_blocks` | Forces `// SAFETY:` comments |

### Documentation Convention

Every `unsafe` block gets a `// SAFETY: ...` comment explaining why it's sound:

```rust
// SAFETY: ptr is non-null and aligned because it came from `Box::into_raw`,
// and we have exclusive access — no other reference exists.
unsafe { Box::from_raw(ptr) }
```

Standard library uses this religiously. Treat it as compiler-enforced docs.

### Common Mistakes

| Mistake | Reality |
|---------|---------|
| Wrapping random code in `unsafe` to silence errors | `unsafe` doesn't disable checks — you'll still get the same error |
| Holding `&mut T` and `&T` simultaneously via pointers | Aliasing violation — UB even if "looks fine" |
| `transmute` between types of different sizes | UB |
| Dereferencing pointer to dropped value | Use-after-free |
| Assuming `unsafe` = "panic on misuse" | UB is silent; it might work today, miscompile tomorrow |

> [!NOTE]
> Goal: keep `unsafe` blocks **small**, **rare**, and **wrapped in safe APIs**. Bad pattern: 200-line `unsafe` block. Good pattern: 3-line `unsafe` with surrounding safe code that establishes preconditions.

### Interview Follow-ups

- *"Why can't the compiler check FFI?"* — Foreign function's invariants aren't expressed in Rust's type system. Compiler can't read its source.
- *"Is `unsafe` Rust still memory-safe?"* — Inside `unsafe`, no — the programmer takes the proof obligation. The rest of the program is safe assuming all `unsafe` blocks are sound.
- *"What's stacked borrows / tree borrows?"* — Aliasing models Miri uses to detect UB. Stacked Borrows is older; Tree Borrows is the proposed successor.
