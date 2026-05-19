# Q: Switch expressions and pattern matching for switch — what changed and why?

**Answer:**

Java's `switch` evolved from a statement that fell through by default into a **typed expression** with arrow labels, exhaustiveness checking, and pattern matching. Today's `switch` replaces visitor pattern, instanceof ladders, and lookup `Map`s for many uses.

### Recap: Classic `switch` Statement

```java
String label;
switch (day) {
    case MONDAY:
    case TUESDAY:
    case WEDNESDAY:
    case THURSDAY:
    case FRIDAY:
        label = "weekday";
        break;
    case SATURDAY:
    case SUNDAY:
        label = "weekend";
        break;
    default:
        throw new IllegalArgumentException();
}
```

Problems:
- `break` is mandatory or fall-through bites.
- Two-step assignment (declare, then assign in each branch).
- No exhaustiveness check from the compiler.
- Can't easily return a value.

### Switch Expression (Java 14+)

```java
String label = switch (day) {
    case MONDAY, TUESDAY, WEDNESDAY, THURSDAY, FRIDAY -> "weekday";
    case SATURDAY, SUNDAY                             -> "weekend";
};
```

Key changes:
- Expression — returns a value, assignable.
- Arrow form — no fall-through, no `break`.
- Comma-separated multiple labels.
- Exhaustive over `enum` → no `default` needed.

### Block Body with `yield`

If a branch needs more than one statement:

```java
int score = switch (grade) {
    case "A" -> 90;
    case "B" -> 80;
    case "C" -> {
        log.info("scraping by");
        yield 70;
    }
    default -> throw new IllegalArgumentException();
};
```

`yield` is the "return from this branch" keyword inside a block.

### Pattern Matching for `switch` (Java 21)

Match by type, deconstruct, and bind in one step.

```java
sealed interface Shape permits Circle, Square, Triangle {}
record Circle(double r) implements Shape {}
record Square(double s) implements Shape {}
record Triangle(double a, double b, double c) implements Shape {}

double area(Shape shape) {
    return switch (shape) {
        case Circle(double r)             -> Math.PI * r * r;
        case Square(double s)             -> s * s;
        case Triangle(double a, double b, double c) -> heron(a, b, c);
    };
}
```

Compiler enforces exhaustiveness because `Shape` is `sealed`. Add a fourth `permits` type? `area` fails to compile.

### Guards

```java
return switch (shape) {
    case Circle c when c.r() == 0 -> 0;
    case Circle c                 -> Math.PI * c.r() * c.r();
    ...
};
```

Order matters — first matching label wins.

### `null` in Switch

Historically, `switch(null)` threw NPE. Now you can match it:

```java
return switch (obj) {
    case null      -> "no value";
    case String s  -> "string: " + s;
    case Integer i -> "int: " + i;
    default        -> "other";
};
```

Without `case null`, NPE still throws — backwards compatibility.

### Combining: Records + Sealed + Switch Patterns

The full Java algebraic style:

```java
sealed interface Result<T> permits Ok, Err {}
record Ok<T>(T value)      implements Result<T> {}
record Err<T>(Throwable e) implements Result<T> {}

<T, R> R fold(Result<T> r, Function<T, R> onOk, Function<Throwable, R> onErr) {
    return switch (r) {
        case Ok<T>(T v)   -> onOk.apply(v);
        case Err<T>(var e) -> onErr.apply(e);
    };
}
```

Reads cleanly; compiler catches missing cases.

### When NOT to Use Switch Expressions

- **Side-effect heavy branches.** A `switch` expression yielding a value with side effects is OK but reads awkwardly; consider a statement form.
- **More than ~5 cases with complex logic per case.** Method dispatch (polymorphism) often reads better.

### Migration Tips

Existing statement → expression:

```java
// Before
switch (op) {
    case PLUS:
        result = a + b;
        break;
    case MINUS:
        result = a - b;
        break;
    default:
        throw new IllegalArgumentException();
}

// After
result = switch (op) {
    case PLUS  -> a + b;
    case MINUS -> a - b;
};
```

Statement form with arrows is allowed too — same fall-through avoidance, just no value:

```java
switch (event) {
    case Click c   -> handleClick(c);
    case KeyPress k -> handleKey(k);
}
```

### Common Mistakes

| Mistake | Reality |
|---------|---------|
| Mixing `case X:` and `case X ->` in one switch | Compile error — must use one style |
| Expecting fall-through with arrows | Doesn't happen; each branch is isolated |
| Missing `default` on non-sealed input | Compile error — switch must be exhaustive |
| Using `yield` in arrow-single-expression form | Wrong; `yield` is only for block form |
| `default` after sealed-exhaustive cases | Permitted but unnecessary; remove to let compiler catch new variants |

### Performance

Switch expressions over enums compile to `tableswitch`/`lookupswitch` bytecode — same as classic switch. Pattern switches over types compile to a chain of `instanceof` checks plus invokedynamic; modern JIT optimizes well. Don't reach for a `Map<K, Function>` for performance — `switch` is at least as fast for the common case.

> [!NOTE]
> Adopt switch expressions in any new code. They eliminate a whole class of bugs (fall-through, missing default) and read better. Pattern matching is the right answer whenever you'd otherwise write a chain of `instanceof`.

### Interview Follow-ups

- *"How does pattern matching handle generics?"* — Type-erasure aware. `case Box<String> b -> ...` is rejected because `Box<String>` and `Box<Integer>` look the same at runtime. Use unchecked casts or design around it.
- *"What is `record` deconstruction?"* — Pulls out components by position: `case Point(int x, int y)` binds `x` and `y`. Works only on `record`s.
- *"Can switch be used in a lambda?"* — Yes, anywhere an expression is allowed. Lambda body can be a switch expression directly.
