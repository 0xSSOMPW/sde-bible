# Q: How do annotations work in Java? Retention, targets, and writing your own.

**Answer:**

Annotations = metadata on classes/methods/fields/parameters/etc. They're declarative — the compiler, framework, or runtime decides what to do with them.

### Built-In Categories
- **Marker** — no members. `@Override`, `@Deprecated`.
- **Single-value** — one element. `@SuppressWarnings("unchecked")`.
- **Multi-value** — multiple elements. `@RequestMapping(path="/x", method=POST)`.
- **Repeating** — same annotation multiple times (Java 8+).
- **Type annotations** — on uses of types, not just declarations (Java 8+). `List<@NotNull String>`.

### Retention (When Annotation Is Available)
```java
@Retention(RetentionPolicy.SOURCE)    // discarded by compiler (e.g., @Override)
@Retention(RetentionPolicy.CLASS)     // in .class file but not at runtime (default)
@Retention(RetentionPolicy.RUNTIME)   // accessible via reflection
```

### Target (Where It Can Be Applied)
```java
@Target({ElementType.TYPE, ElementType.METHOD, ElementType.FIELD,
         ElementType.PARAMETER, ElementType.CONSTRUCTOR,
         ElementType.LOCAL_VARIABLE, ElementType.ANNOTATION_TYPE,
         ElementType.PACKAGE, ElementType.TYPE_PARAMETER, ElementType.TYPE_USE,
         ElementType.MODULE, ElementType.RECORD_COMPONENT})
```

### Custom Annotation Skeleton
```java
import java.lang.annotation.*;

@Target({ElementType.METHOD, ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Documented              // appears in Javadoc
@Inherited               // subclasses inherit (only ElementType.TYPE)
public @interface Auditable {
    String action() default "";
    String[] tags() default {};
    boolean async() default false;
}
```

Usage:
```java
@Auditable(action = "ORDER_CREATE", tags = {"orders","write"})
public Order create(...) { ... }
```

### Element Types Allowed
- Primitives, `String`, `Class`, enum, annotation, **arrays of those**.
- Not arbitrary objects.

### Reading at Runtime
```java
Method m = OrderService.class.getMethod("create", ...);
if (m.isAnnotationPresent(Auditable.class)) {
    Auditable a = m.getAnnotation(Auditable.class);
    System.out.println(a.action());
}
```

### Repeating Annotations (Java 8+)
```java
@Repeatable(Schedules.class)
public @interface Schedule { String cron(); }

public @interface Schedules { Schedule[] value(); }

@Schedule(cron="0 0 * * * *")
@Schedule(cron="0 0 12 * * *")
public void run() { }
```

### Type Annotations (Java 8+)
```java
public @NonNull String greet(@NonNull String name) { ... }
List<@NonNull String> names;
String s = (@NonNull String) obj;
```
Used by tools like Checker Framework for null-safety.

### Meta-Annotations (Annotations on Annotations)
You build composite annotations:

```java
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
@Transactional(propagation = Propagation.REQUIRES_NEW)
@Auditable
public @interface DomainOperation { }
```

Spring follows this pattern — `@RestController`, `@Service`, `@Repository` are all meta-annotated with `@Component`.

### Spring's Common Annotation Categories
| Category | Examples |
|---|---|
| Stereotype | `@Component`, `@Service`, `@Repository`, `@Controller` |
| Wiring | `@Autowired`, `@Qualifier`, `@Value`, `@Lazy` |
| Config | `@Configuration`, `@Bean`, `@Profile`, `@ConditionalOn*` |
| Web | `@RestController`, `@GetMapping`, `@RequestBody`, `@PathVariable` |
| Data/Tx | `@Transactional`, `@Entity`, `@Id`, `@Query` |
| AOP | `@Aspect`, `@Before`, `@Around` |
| Validation | `@NotNull`, `@Size`, `@Valid`, `@Validated` |

### Annotation Processors (Compile-Time)
`javax.annotation.processing` API. Read `SOURCE` / `CLASS`-retention annotations during compilation and generate code/reports.

Famous users:
- **Lombok** — generates getters, setters, `equals`, `hashCode`, etc.
- **MapStruct** — generates DTO ↔ entity mappers.
- **Dagger** — DI graph code generation.
- **AutoValue** — value-class generation.
- **Hibernate Metamodel** — type-safe Criteria API metamodel.

Custom processor:
```java
@SupportedAnnotationTypes("com.acme.GenerateBuilder")
@SupportedSourceVersion(SourceVersion.RELEASE_17)
public class BuilderProcessor extends AbstractProcessor {
    @Override
    public boolean process(Set<? extends TypeElement> annotations, RoundEnvironment env) {
        for (Element e : env.getElementsAnnotatedWith(GenerateBuilder.class)) {
            // generate code via Filer
        }
        return true;
    }
}
```

### When To Write A Custom Annotation
- **Marker** for cross-cutting behavior + a Spring aspect / interceptor (`@RateLimit`, `@Audited`).
- **Configuration switch** read by a runtime framework you control.
- **Compile-time generation** (annotation processor).

### When NOT To
- Hiding logic that should be obvious in code.
- Replacing what a method or interface would express better.
- "Magic" that obscures program flow without clear benefit.

### Default Methods on Annotations? Defaults Yes; Methods No
Annotations are interfaces under the hood, but you can only declare element methods (no body, no default behavior beyond `default` values).

### Pitfalls
- Forgetting `@Retention(RUNTIME)` → annotation invisible at runtime.
- Forgetting to `@Inherited` and assuming subclasses pick up the annotation (only works on `TYPE`-level).
- Annotation present but no processor / aspect to act on it → silent no-op.
- Putting heavy logic (string parsing, regex compilation) in annotation users — cache the parsed form.
