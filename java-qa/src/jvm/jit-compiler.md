# Q: How does the JIT compiler work? What are C1/C2 and tiered compilation?

**Answer:**

JVM starts by **interpreting** bytecode. Hot methods are then **JIT-compiled** to native code. JIT = Just-In-Time.

### Why Not AOT-compile Everything?
- Startup latency.
- Profile-guided optimization needs runtime data (which branches taken, types seen).
- AOT can't speculate; JIT can (and de-optimize on guess wrong).

### Two Compilers
- **C1 (client)**: fast compile, modest optimization.
- **C2 (server)**: slow compile, aggressive optimization (inlining, escape analysis, vectorization).

### Tiered Compilation (Default Since JDK 8)
| Tier | Who | Profiling | Speed |
|---|---|---|---|
| 0 | Interpreter | yes | slowest |
| 1 | C1 (no profiling) | no | fast compile, fast run |
| 2 | C1 (limited profiling) | partial | |
| 3 | C1 (full profiling) | full | |
| 4 | C2 / Graal | uses tier-3 profile | slowest compile, fastest run |

Hot path: 0 → 3 → 4. Cold quick wins: 0 → 1.

### Triggers
- Invocation count + back-edge (loop) count crosses thresholds (`-XX:CompileThreshold=10000` for non-tiered).
- "Hot" = called often or contains hot loop.

### Key Optimizations
- **Inlining** — replace method call with body. Enables further optimization.
- **Escape analysis** — if object never escapes a method, allocate on stack or scalar-replace.
- **Lock elision** — remove locks on objects proven thread-local.
- **Loop unrolling** + vectorization (SIMD).
- **Branch prediction hints** from profile.
- **Devirtualization** — turn virtual call into direct/inlined call when only one impl observed.

### De-optimization
JIT speculates ("only seen `ArrayList` here"). When assumption breaks ("now a `LinkedList` showed up"), JVM throws away compiled code, falls back to interpreter, recompiles with new info.

### Useful Flags
```bash
-XX:+PrintCompilation              # log JIT events
-XX:+UnlockDiagnosticVMOptions -XX:+PrintInlining
-XX:CICompilerCount=4              # JIT compiler threads
-XX:+TieredCompilation             # default on
-XX:TieredStopAtLevel=1            # disable C2 — faster startup, slower steady state
-XX:+UseCodeCacheFlushing
```

### GraalVM
Drop-in replacement for C2, written in Java. Often faster on polyglot/Scala/Kotlin code.
```
-XX:+UnlockExperimentalVMOptions -XX:+UseJVMCICompiler
```

### AOT Options
- **`jaotc`** (deprecated/removed) — AOT-compile classes.
- **GraalVM Native Image** — full AOT, ~ms startup, lower peak throughput, no JIT speculation.

### Code Cache
Compiled native code lives in code cache (separate from heap). Fills up → JIT stops → "CodeCache is full. Compiler has been disabled" log warning. Bump with `-XX:ReservedCodeCacheSize=256m`.

### Warmup
Benchmarks must include warmup loop — first invocations run interpreted. Use **JMH** for microbenchmarks; it handles warmup correctly.

### Interview Soundbite
> "JIT means hot code becomes fast over time, cold code stays cheap. C1 prioritizes compile speed, C2 prioritizes runtime speed. Tiered compilation runs both — interpret first, C1 for warmup, C2 for steady state. Speculation enables aggressive optimization with de-opt as the safety net."
