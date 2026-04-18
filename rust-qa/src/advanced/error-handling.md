# Q: How do you handle errors in Rust?

**Answer:**

Rust groups errors into two major categories: **Recoverable** and **Unrecoverable** errors. 

### 1. Recoverable Errors (`Result<T, E>`)

For errors that can be handled gracefully (like a file not being found), Rust uses the `Result` enum:

```rust
enum Result<T, E> {
    Ok(T),
    Err(E),
}
```

You can handle `Result` using `match` or the `?` operator.

**Using `match`:**
```rust
use std::fs::File;

let f = File::open("hello.txt");
let f = match f {
    Ok(file) => file,
    Err(error) => panic!("Problem opening the file: {:?}", error),
};
```

**Using the `?` Operator:**
The `?` operator is a shorthand that unwraps `Ok` values or immediately returns the `Err` from the current function.
```rust
use std::fs::File;
use std::io::{self, Read};

fn read_username_from_file() -> Result<String, io::Error> {
    let mut f = File::open("hello.txt")?;
    let mut s = String::new();
    f.read_to_string(&mut s)?;
    Ok(s)
}
```

### 2. Unrecoverable Errors (`panic!`)

For situations where the program reaches a state that it cannot recover from (like accessing an array out of bounds), Rust provides the `panic!` macro. When a panic occurs, the program will print a failure message, unwind and clean up the stack, and then quit.

```rust
fn main() {
    panic!("crash and burn");
}
```
