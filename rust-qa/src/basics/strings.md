# Q: What is the difference between `String` and `&str` in Rust?

**Answer:**

In Rust, `String` and `&str` are both used to handle string data, but they have distinct differences in how they manage memory and handle ownership:

1.  **`String`**:
    *   It is an **owned** type.
    *   It is stored on the **heap**, meaning its size can grow or shrink at runtime.
    *   When a `String` goes out of scope, its memory is automatically freed (dropped).
    *   You can mutate it (if it's declared `mut`), e.g., by pushing new characters or strings to it.

2.  **`&str` (String Slice)**:
    *   It is a **borrowed** type (a reference).
    *   It represents a view into a block of memory that contains a string (which could be on the heap, stack, or hardcoded in the binary as a static string).
    *   It does not have ownership of the data it points to; it merely looks at it temporarily.
    *   It is immutable by default and its size is fixed.

**Example:**
```rust
fn main() {
    // A String (heap-allocated, owned, growable)
    let mut my_string = String::from("Hello");
    my_string.push_str(", world!");

    // A &str (string slice, borrowed, fixed-size view)
    let my_slice: &str = &my_string[0..5]; // Borrows "Hello"
    
    // Hardcoded string literals are also of type &str (specifically &'static str)
    let static_str: &str = "I am stored in the binary";
}
```
