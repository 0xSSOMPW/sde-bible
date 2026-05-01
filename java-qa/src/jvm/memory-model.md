# Q: Explain the JVM Memory Model.

**Answer:**

### JVM Memory Areas

```
┌──────────────────────────────────────────┐
│                JVM Memory                │
│                                          │
│  ┌──────────────────────────────────┐    │
│  │           HEAP                   │    │
│  │  ┌────────────┐  ┌───────────┐  │    │
│  │  │ Young Gen  │  │  Old Gen  │  │    │
│  │  │ ┌────────┐ │  │(Tenured)  │  │    │
│  │  │ │  Eden  │ │  │           │  │    │
│  │  │ ├────────┤ │  │  Long-    │  │    │
│  │  │ │  S0    │ │  │  lived    │  │    │
│  │  │ │  S1    │ │  │  objects  │  │    │
│  │  │ └────────┘ │  │           │  │    │
│  │  └────────────┘  └───────────┘  │    │
│  └──────────────────────────────────┘    │
│                                          │
│  ┌──────────┐  ┌─────────────────────┐   │
│  │  Stack   │  │   Metaspace         │   │
│  │ (per     │  │ (class metadata,    │   │
│  │  thread) │  │  method bytecode)   │   │
│  └──────────┘  └─────────────────────┘   │
└──────────────────────────────────────────┘
```

### Heap (Shared Across All Threads)
Where **objects** live. Divided into generations for GC efficiency:

- **Young Generation**: Newly created objects. Most objects die here (short-lived).
  - **Eden**: Objects are initially allocated here.
  - **Survivor Spaces (S0, S1)**: Objects that survive minor GC move here.
- **Old Generation (Tenured)**: Objects that survive multiple minor GC cycles are promoted here. Full GC cleans this area.

### Stack (Per Thread)
Each thread has its own stack storing:
- **Stack frames**: One per method call, containing local variables, method parameters, and return addresses.
- Primitive values and object references (not the objects themselves).
- Fixed size: too many frames = `StackOverflowError`.

### Metaspace (Java 8+, replaces PermGen)
Stores **class metadata**, method bytecode, constant pool, and static variables. Uses **native memory** (not heap), so it auto-grows (configurable with `-XX:MaxMetaspaceSize`).

### Other Areas
- **PC Register**: Per thread, tracks the current bytecode instruction.
- **Native Method Stack**: For JNI native method calls.

### Key JVM Flags

```bash
java -Xms512m     # Initial heap size
     -Xmx2g       # Maximum heap size
     -Xss1m       # Thread stack size
     -XX:MetaspaceSize=256m
     -XX:MaxMetaspaceSize=512m
     -XX:+PrintGCDetails  # Log GC activity
```

> [!IMPORTANT]
> **Stack vs Heap**: Primitives and references are stored on the **stack** (fast, per-thread). Objects are stored on the **heap** (shared, GC-managed). This distinction is fundamental to understanding memory management, thread safety, and performance tuning.
