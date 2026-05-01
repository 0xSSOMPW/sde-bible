# Q: How does Exception Handling work in Java? Checked vs Unchecked?

**Answer:**

### Exception Hierarchy

```
        Throwable
        /       \
    Error     Exception
              /       \
  Checked Exceptions   RuntimeException (Unchecked)
  (IOException,        (NullPointerException,
   SQLException)        IllegalArgumentException,
                        ArrayIndexOutOfBoundsException)
```

### Checked Exceptions
Checked at **compile time**. The compiler forces you to either `catch` them or declare them with `throws`. They represent **recoverable** conditions.

```java
// Must handle or declare
public void readFile() throws IOException {
    FileReader reader = new FileReader("data.txt"); // IOException is checked
}
```

**Examples:** `IOException`, `SQLException`, `ClassNotFoundException`

### Unchecked Exceptions (RuntimeException)
NOT checked at compile time. They represent **programming bugs** that shouldn't be caught with a catch block — they should be fixed in the code.

```java
String s = null;
s.length();  // NullPointerException — unchecked, no compile error
```

**Examples:** `NullPointerException`, `ArrayIndexOutOfBoundsException`, `ClassCastException`, `IllegalArgumentException`

### Errors
Represent **unrecoverable JVM-level problems**. You should NOT catch these.

**Examples:** `OutOfMemoryError`, `StackOverflowError`, `VirtualMachineError`

### try-with-resources (Java 7+)
Automatically closes resources that implement `AutoCloseable`:

```java
// ❌ Old style: verbose, error-prone
BufferedReader reader = null;
try {
    reader = new BufferedReader(new FileReader("file.txt"));
    String line = reader.readLine();
} catch (IOException e) {
    e.printStackTrace();
} finally {
    if (reader != null) {
        try { reader.close(); } catch (IOException e) { /* swallowed */ }
    }
}

// ✅ try-with-resources: auto-closes, clean
try (BufferedReader reader = new BufferedReader(new FileReader("file.txt"))) {
    String line = reader.readLine();
} catch (IOException e) {
    e.printStackTrace();
}
// reader.close() is called automatically, even if an exception occurs
```

### Multi-catch (Java 7+)
```java
try {
    // ...
} catch (IOException | SQLException e) {
    log.error("Failed", e);
}
```

> [!TIP]
> In interviews, a strong take is: "I prefer unchecked exceptions for application-level errors with clear documentation, and checked exceptions only at API boundaries where the caller genuinely needs to handle the failure mode." This shows you understand the ongoing debate in the Java community about checked vs unchecked exception design.
