# Q: What is the difference between JDK, JRE, and JVM?

**Answer:**

This is the most common Java interview opener. These three form a layered architecture.

### JVM (Java Virtual Machine)
The JVM is an **abstract machine** that provides the runtime environment to execute Java bytecode. It does NOT understand Java source code — only `.class` bytecode files.

**Responsibilities:**
- Loading bytecode (via ClassLoader)
- Verifying bytecode (bytecode verifier)
- Executing bytecode (interpreter + JIT compiler)
- Managing memory (heap, stack, garbage collection)

**Key point:** The JVM is what makes Java **platform-independent**. The same `.class` file runs on any OS that has a JVM implementation (Windows, macOS, Linux).

### JRE (Java Runtime Environment)
The JRE = **JVM + standard class libraries** (java.lang, java.util, java.io, etc.). It's everything you need to **run** a Java application, but you cannot compile code with it.

### JDK (Java Development Kit)
The JDK = **JRE + development tools** (javac compiler, javadoc, jdb debugger, jconsole, etc.). It's what developers install to **develop and compile** Java applications.

### The Relationship

```
┌───────────────────────────────────┐
│             JDK                   │
│  ┌─────────────────────────────┐  │
│  │           JRE               │  │
│  │  ┌───────────────────────┐  │  │
│  │  │         JVM           │  │  │
│  │  │  (bytecode execution) │  │  │
│  │  └───────────────────────┘  │  │
│  │  + Standard Libraries       │  │
│  │    (rt.jar, java.*, etc.)   │  │
│  └─────────────────────────────┘  │
│  + Development Tools              │
│    (javac, javadoc, jar, jdb)     │
└───────────────────────────────────┘
```

> [!NOTE]
> Since Java 11, Oracle no longer ships a separate JRE. The JDK is the only downloadable package, and you can create custom minimal runtimes using `jlink`.
