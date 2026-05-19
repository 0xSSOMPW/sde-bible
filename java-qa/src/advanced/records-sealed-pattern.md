# Q: What are records, sealed classes, and pattern matching in modern Java?

**Answer:**

These three Java 16–21 features together push Java toward a more **algebraic-data-type** style: small immutable data carriers (records), closed type hierarchies (sealed), and exhaustive structural deconstruction (pattern matching). Used together they replace a lot of boilerplate visitor/equals/hashCode/instanceof code.

### Records (Java 16)

A record is a class whose entire purpose is to be a transparent carrier of values.

```java
public record Point(int x, int y) {}
```

The compiler generates:
- A canonical constructor.
- `final` fields `x`, `y`.
- Accessors `x()`, `y()` (no `get` prefix).
- `equals`, `hashCode`, `toString` based on the components.

Override behavior selectively:

```java
public record Range(int lo, int hi) {
    // Compact constructor — validate without re-declaring parameters.
    public Range {
        if (lo > hi) throw new IllegalArgumentException();
    }
}
```

Records can implement interfaces and declare static methods, but **cannot extend a class** (they implicitly extend `java.lang.Record`).

When NOT a record:
- Mutable state required.
- Inheritance from a base class.
- Identity matters (you want reference equality).

### Sealed Classes (Java 17)

`sealed` restricts which types can extend/implement a type. The hierarchy is closed and known at compile time.

```java
public sealed interface Shape permits Circle, Square, Triangle {}

public record Circle(double radius) implements Shape {}
public record Square(double side) implements Shape {}
public record Triangle(double a, double b, double c) implements Shape {}
```

Permitted subtypes must be declared `final`, `sealed`, or `non-sealed`. Records are implicitly `final`, so they fit naturally.

Why it matters: the compiler now **knows** the full set of subtypes. Combined with pattern matching, you get exhaustiveness checking.

### Pattern Matching for `instanceof` (Java 16)

```java
// Old
if (obj instanceof String) {
    String s = (String) obj;
    return s.length();
}

// New — binding variable
if (obj instanceof String s) {
    return s.length();
}
```

### Pattern Matching for `switch` (Java 21)

```java
double area(Shape shape) {
    return switch (shape) {
        case Circle c       -> Math.PI * c.radius() * c.radius();
        case Square s       -> s.side() * s.side();
        case Triangle t     -> heron(t.a(), t.b(), t.c());
        // No default — compiler verifies exhaustiveness because Shape is sealed.
    };
}
```

If you add a fourth permitted subtype, this method **fails to compile** until you handle it. That's the safety win: the type system enforces case coverage.

### Record Deconstruction Patterns (Java 21)

You can destructure inside `instanceof`/`switch`:

```java
record Pair(int x, int y) {}

if (obj instanceof Pair(int x, int y)) {
    return x + y;
}

return switch (shape) {
    case Circle(double r)             -> Math.PI * r * r;
    case Square(double s)             -> s * s;
    case Triangle(double a, double b, double c) -> heron(a, b, c);
};
```

Nested patterns:

```java
record Line(Point a, Point b) {}

case Line(Point(int x1, int y1), Point(int x2, int y2)) ->
    Math.hypot(x2 - x1, y2 - y1);
```

### Guards (`when`)

```java
return switch (shape) {
    case Circle c when c.radius() == 0   -> 0;
    case Circle c                        -> Math.PI * c.radius() * c.radius();
    ...
};
```

Order matters: more specific guards first.

### Putting It Together

```java
sealed interface Json permits JNull, JBool, JNum, JStr, JArr, JObj {}
record JNull()                            implements Json {}
record JBool(boolean v)                   implements Json {}
record JNum(double v)                     implements Json {}
record JStr(String v)                     implements Json {}
record JArr(List<Json> v)                 implements Json {}
record JObj(Map<String, Json> v)          implements Json {}

String stringify(Json j) {
    return switch (j) {
        case JNull n          -> "null";
        case JBool(boolean v) -> Boolean.toString(v);
        case JNum(double v)   -> Double.toString(v);
        case JStr(String v)   -> "\"" + v + "\"";
        case JArr(List<Json> v) -> v.stream().map(this::stringify).collect(joining(",", "[", "]"));
        case JObj(Map<String,Json> v) ->
            v.entrySet().stream()
                .map(e -> "\"" + e.getKey() + "\":" + stringify(e.getValue()))
                .collect(joining(",", "{", "}"));
    };
}
```

No visitor pattern, no class-explosion, exhaustively type-checked.

### Common Mistakes

| Mistake | Reality |
|---------|---------|
| Treating records as Lombok replacements universally | Lombok generates getters/setters; records are *immutable*. Use only where immutability fits |
| Long parameter lists in record components | A record with 12 components is a smell; group sub-records |
| Sealing a public API hierarchy then needing to add a type | Sealed is a *commitment*. Use only when the set is genuinely closed |
| Forgetting that `default` in switch breaks exhaustiveness checking | Omit `default` for sealed switches — let the compiler do its job |

> [!NOTE]
> These features are most valuable *together*: sealed defines the algebra; records carry data; pattern matching consumes them. Used piecemeal they look like ceremony; used together they replace whole patterns.

### Interview Follow-ups

- *"Can a record be mutable?"* — Components are final. But components can be mutable types (`record R(List<String> items)`) — best to wrap in `List.copyOf` in a compact constructor.
- *"Difference between sealed and final?"* — `final` = no subtypes. `sealed` = a known, enumerated set of subtypes. `non-sealed` opens that branch back up.
- *"Will pattern matching support arrays/Maps?"* — Array patterns are previewed; map patterns aren't standardized yet (JEP under discussion).
