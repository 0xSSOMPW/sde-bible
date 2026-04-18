# Q: Explain Ownership and Borrowing in Rust.

**Answer:**

**Ownership** is Rust's most unique feature, which guarantees memory safety without needing a garbage collector. It operates on three main rules:
1.  Each value in Rust has a single variable that is its **owner**.
2.  There can only be one owner at a time.
3.  When the owner goes out of scope, the value is dropped (its memory is freed).

**Borrowing** is how Rust allows you to access a value without taking ownership of it, using references (`&` or `&mut`). It solves the problem of needing to pass values to functions without losing ownership.

Borrowing has two strict rules:
1.  At any given time, you can have *either* **one mutable reference** (`&mut T`) *or* **any number of immutable references** (`&T`).
2.  References must always be valid (Rust prevents dangling pointers).

**Example of Ownership:**
```rust
let s1 = String::from("hello");
let s2 = s1; // Ownership moves to s2
// println!("{}", s1); // Error! s1 is no longer valid
```

**Example of Borrowing:**
```rust
fn calculate_length(s: &String) -> usize { // Takes an immutable reference
    s.len()
} // s goes out of scope, but since it doesn't have ownership, nothing is dropped.

let s1 = String::from("hello");
let len = calculate_length(&s1); // We borrow s1 instead of moving it
```
