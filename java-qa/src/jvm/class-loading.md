# Q: How does ClassLoading work in Java?

**Answer:**

Class loading is the process of finding, loading, and initializing `.class` files into the JVM.

### The Three Phases

**1. Loading** — The ClassLoader reads the `.class` bytecode file and creates a `Class<?>` object in memory.

**2. Linking**
- **Verification**: Bytecode is checked for correctness and security.
- **Preparation**: Static fields are allocated and set to default values (0, null).
- **Resolution**: Symbolic references (class names in bytecode) are resolved to actual memory addresses.

**3. Initialization** — Static initializers and static blocks are executed. This happens only when the class is **first used**.

### ClassLoader Hierarchy (Delegation Model)

```
Bootstrap ClassLoader (C/C++)
    ↑ delegates to parent first
Application ClassLoader
    ↑
Extension ClassLoader
    ↑
Custom ClassLoader (your code)
```

| ClassLoader | Loads From | Examples |
|---|---|---|
| **Bootstrap** | `$JAVA_HOME/lib` (core classes) | `java.lang.String`, `java.util.*` |
| **Extension/Platform** | `$JAVA_HOME/lib/ext` | Security, crypto extensions |
| **Application/System** | Classpath (`-cp`, `CLASSPATH`) | Your application classes |
| **Custom** | Anywhere you define | Plugin systems, hot-reloading |

### Parent-Delegation Model
When a class needs to be loaded:
1. The current ClassLoader delegates to its **parent** first.
2. If the parent can't find it, the current ClassLoader tries.
3. If no one can find it → `ClassNotFoundException`.

**Why?** Prevents duplicate class loading and ensures core classes (`java.lang.String`) are always loaded by the Bootstrap ClassLoader, preventing tampering.

### Common Interview Scenarios

```java
// These are loaded by DIFFERENT classloaders:
String.class.getClassLoader();    // null (Bootstrap — implemented in native code)
MyApp.class.getClassLoader();     // AppClassLoader
```

**ClassNotFoundException vs NoClassDefFoundError:**

| Exception | Cause |
|---|---|
| `ClassNotFoundException` | Class not found at runtime (e.g., `Class.forName("Missing")`) |
| `NoClassDefFoundError` | Class was available at compile time but missing at runtime |

> [!NOTE]
> Understanding class loading is essential for debugging issues in application servers (Tomcat, Spring Boot), OSGi frameworks, and anywhere with multiple classloaders. "Class X cannot be cast to Class X" errors typically mean the same class was loaded by two different classloaders.
