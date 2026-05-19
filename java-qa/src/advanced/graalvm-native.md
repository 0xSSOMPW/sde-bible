# Q: What is GraalVM Native Image, and when should you use it?

**Answer:**

GraalVM Native Image **AOT-compiles** a Java application into a standalone native binary. Startup goes from seconds to milliseconds, memory drops 5–10×, but you lose JIT peak throughput and reflection-driven libraries need extra configuration.

### What It Produces

```
javac + JIT (HotSpot):
  app.jar  +  JDK  +  JIT warmup  →  fast peak throughput, slow startup, big RAM

native-image:
  app.jar  →  app  (one ELF/Mach-O binary)  →  fast startup, small RAM, lower peak
```

Output is a single executable with:
- All reachable JDK + library + app classes compiled to native code.
- Pre-initialized heap snapshot.
- No JVM, no class loading at runtime.

### Closed-World Assumption

GraalVM analyzes the entire program at build time. Everything reachable must be **statically discoverable**. Code paths reached only via reflection, JNI, proxies, resource loading, or serialization need explicit registration.

```
Source files + classpath
       │
       ▼  static analysis (points-to)
   reachable methods, classes, fields
       │
       ▼  AOT compile (Graal)
       native binary
```

The trade: smaller, faster-starting binary; less runtime flexibility.

### Building It

```bash
# Install GraalVM (with native-image component)
sdk install java 21.0.2-graal

# From a Spring Boot project:
./mvnw -Pnative native:compile

# Run:
./target/app
```

Generic:

```bash
native-image -jar app.jar -o app \
  --no-fallback \
  -O3 \
  --gc=G1
```

### Startup and Memory Comparison

Typical Spring Boot REST service:

| Metric | JVM | Native Image |
|--------|-----|--------------|
| Startup time | 2.5 s | 40 ms |
| First-request latency | varies (JIT warmup) | constant |
| RSS (idle) | 250 MB | 35 MB |
| Peak throughput | 100% | 70–90% |
| Image size | 50 MB JAR + JRE | 80–120 MB binary |
| Build time | 10 s | 90 s — 5 min |

### Where Native Image Wins

- **Serverless functions** (AWS Lambda, GCP Cloud Functions). Cold start dominates user-perceived latency.
- **Containers with autoscaling.** New pods ready in 100 ms, not 5 s.
- **CLI tools written in Java.** No JVM startup tax.
- **Memory-constrained environments.** 10× less RSS = more pods per node.

### Where It Loses

- **Long-running CPU-bound services** where JIT eventually beats AOT.
- **Apps heavy on reflection/proxies** without good framework support.
- **Build-time CI cost** matters (5-minute native build vs 30-second JAR build).

### Reflection Configuration

```java
Class.forName("com.example.Plugin");   // reflective load
```

Graal's analyzer doesn't see this. You declare it in `reflect-config.json`:

```json
[
  {
    "name": "com.example.Plugin",
    "allDeclaredConstructors": true,
    "allPublicMethods": true
  }
]
```

Generate automatically by running the app on the **tracing agent** first:

```bash
java -agentlib:native-image-agent=config-output-dir=./meta -jar app.jar
# Exercise all code paths via tests
# Outputs reflect-config.json, resource-config.json, jni-config.json, proxy-config.json
```

Spring AOT plugin, Quarkus, and Micronaut do this automatically — that's their main value proposition.

### Frameworks Ranked by Native Support

| Framework | Native Story |
|-----------|--------------|
| **Quarkus** | Built for native from day 1. Best UX. |
| **Micronaut** | AOT-by-design, no runtime reflection. Excellent. |
| **Spring Boot 3+** | First-class via `spring-aot`. Works well; not all starters supported equally. |
| **Helidon SE** | Native-first. Good. |
| **Plain Spring (pre-3.0) / Hibernate** | Possible but painful. |

### Build-Time vs Run-Time Initialization

By default, class `<clinit>` blocks run **at image build time** and the resulting state is captured in the image heap. This is what makes startup fast — but it can break things:

```java
// Calls happen at BUILD time on the build machine
class Config {
    static final String HOSTNAME = InetAddress.getLocalHost().getHostName();
}
```

Now every deployed instance reports the build machine's hostname. Mark it for runtime init:

```bash
native-image --initialize-at-run-time=com.example.Config ...
```

Conversely, force init at build time for performance:

```bash
--initialize-at-build-time=com.example.Static
```

### Profile-Guided Optimization (PGO)

GraalVM Enterprise (and community 23+) supports PGO:

```bash
# 1. Build instrumented binary
native-image --pgo-instrument -jar app.jar
./app          # exercise workload
# default.iprof generated

# 2. Build optimized binary using the profile
native-image --pgo=default.iprof -jar app.jar
```

PGO recovers most of the peak-throughput gap to JIT.

### Garbage Collection

Native Image ships with:

- **Serial GC** (default): single-threaded, low memory overhead, good for serverless/short jobs.
- **G1**: production-grade concurrent GC (Linux x64 only in community).
- **Epsilon**: no-op GC for short-lived programs.

```bash
native-image --gc=G1 -jar app.jar
```

### Foreign Code (JNI, JFR, Agents)

- **JNI**: works, but each native call is configured in `jni-config.json`.
- **JFR (Flight Recorder)**: supported in newer versions; profile production binaries.
- **Java agents**: build-time only; runtime instrumentation isn't possible.

### Build Image Size Optimization

```bash
native-image \
  --no-fallback \
  -O3 \
  -H:Optimize=3 \
  --enable-preview \
  -H:+ReportExceptionStackTraces \
  -H:IncludeResources='.*\.(properties|yaml|xml|json|sql)' \
  -jar app.jar
```

UPX further compresses (cost: startup +20 ms decompressing).

### Common Mistakes

| Mistake | Fix |
|---------|-----|
| Reflection used by a library; "ClassNotFoundException at runtime" | Run tracing agent, commit generated config |
| Static block reads env at build time | `--initialize-at-run-time=` |
| Bloated binary with all resources included | Tighten the `IncludeResources` regex |
| `Class.forName` on user-supplied strings | Doesn't work; refactor or maintain a closed list |
| Logging slow at startup | Pre-initialize logger at build time |

> [!NOTE]
> Native Image is not a free upgrade — it's a different deployment model. Treat the AOT build as a *separate artifact* with its own test suite. CI: run integration tests against the native binary, not just the JVM build.

### Interview Follow-ups

- *"Why is reflection a problem?"* — Static reachability analysis can't see "open this class by string name at runtime."
- *"How does this differ from `jlink`?"* — `jlink` produces a custom JRE (still JVM-based). `native-image` produces a JVM-less binary.
- *"What's `Substrate VM`?"* — The runtime inside a native image — a small VM with its own GC, scheduler, exception handling. About 10 MB of code.
