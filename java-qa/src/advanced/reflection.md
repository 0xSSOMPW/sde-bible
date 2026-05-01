# Q: How does Java reflection work? When to use it, what are the costs?

**Answer:**

Reflection = inspect + manipulate classes/methods/fields at runtime. The class metadata in the JVM is exposed via `java.lang.reflect`.

### Basic Operations
```java
Class<?> c = Class.forName("com.acme.User");
// or User.class, or user.getClass()

// Inspect
c.getDeclaredFields();
c.getDeclaredMethods();
c.getDeclaredConstructors();
c.getInterfaces();
c.getSuperclass();
c.isAnnotationPresent(Entity.class);

// Instantiate
Constructor<?> ctor = c.getDeclaredConstructor(String.class, int.class);
Object instance = ctor.newInstance("alice", 30);

// Invoke method
Method m = c.getDeclaredMethod("greet", String.class);
m.setAccessible(true);                  // bypass private
Object result = m.invoke(instance, "world");

// Read/write field
Field f = c.getDeclaredField("name");
f.setAccessible(true);
f.set(instance, "bob");
String name = (String) f.get(instance);
```

### Annotation Reading
```java
for (Method m : c.getDeclaredMethods()) {
    if (m.isAnnotationPresent(MyAnnotation.class)) {
        MyAnnotation ann = m.getAnnotation(MyAnnotation.class);
        System.out.println(ann.value());
    }
}
```

### Where Reflection Powers The Java Ecosystem
- **Spring** — bean instantiation, DI, `@Autowired` field injection, AOP proxies.
- **Hibernate / JPA** — entity field access, lazy proxies.
- **Jackson / Gson** — serialize/deserialize without manual mappings.
- **JUnit / TestNG** — discover `@Test` methods.
- **Mockito** — mock generation.
- **Logging frameworks**, ORM, IoC, validators (Bean Validation), serializers, deserializers, ...

### Costs

**1. Performance**
Reflection is slower than direct calls. JIT can optimize repeated reflective calls (caching `MethodAccessor`), but not as well as direct invocation.

Rough rule of thumb (varies, measure for your case):
- Direct call: ~1ns
- Cached `Method.invoke`: ~10-50ns
- Uncached: 100s of ns to µs

Avoid in tight loops. Cache `Method`/`Field` references.

**2. No compile-time safety**
Method names are strings → typos blow up at runtime, not compile time.

**3. Strong encapsulation (Java 9+)**
JPMS modules + `--illegal-access` controls block deep reflection on JDK internals. Setting `setAccessible(true)` on private members of other modules requires the module to `opens` the package.

```
Add-Opens=java.base/java.lang=ALL-UNNAMED   # JAR manifest, e.g., for older libs
```

**4. Security**
Bypassing `private` violates encapsulation contracts. Avoid in production code.

### Modern Alternatives

**1. `MethodHandle` (Java 7+)**
Faster than reflection. JIT-friendly.
```java
MethodHandles.Lookup lookup = MethodHandles.lookup();
MethodHandle mh = lookup.findVirtual(User.class, "greet", MethodType.methodType(String.class, String.class));
String s = (String) mh.invokeExact(user, "world");
```

**2. `VarHandle` (Java 9+)**
For fields. Replaces `sun.misc.Unsafe` for atomic operations.

**3. Annotation processors / code generation**
Lombok, MapStruct, Dagger: generate code at compile time → no runtime reflection cost.

**4. Records + pattern matching**
Reduces need for reflective deconstruction.

### Real Reflection Examples

**Generic factory**
```java
public static <T> T newInstance(Class<T> c) {
    try { return c.getDeclaredConstructor().newInstance(); }
    catch (Exception e) { throw new RuntimeException(e); }
}
```

**Find all fields with annotation**
```java
List<Field> idFields = Arrays.stream(c.getDeclaredFields())
    .filter(f -> f.isAnnotationPresent(Id.class))
    .toList();
```

**Dynamic proxy (no aspect framework needed)**
```java
@SuppressWarnings("unchecked")
public static <T> T loggingProxy(T target, Class<T> iface) {
    return (T) Proxy.newProxyInstance(
        iface.getClassLoader(),
        new Class<?>[]{ iface },
        (proxy, method, args) -> {
            System.out.println("calling " + method.getName());
            return method.invoke(target, args);
        });
}
```

### Generic Type Erasure + Reflection
Generic types erased at runtime, but **declared types** preserved on fields, methods, classes:
```java
Field f = User.class.getDeclaredField("orders");      // List<Order> orders;
ParameterizedType pt = (ParameterizedType) f.getGenericType();
Class<?> actualType = (Class<?>) pt.getActualTypeArguments()[0];  // Order.class
```

### Best Practices
- Cache `Method`, `Field`, `Constructor` lookups.
- Catch and wrap checked exceptions sensibly.
- Prefer `MethodHandle` over `Method.invoke` in hot paths.
- Prefer compile-time generation (annotation processors) over runtime reflection.
- Don't use reflection to break encapsulation in your own code — it's for frameworks.
