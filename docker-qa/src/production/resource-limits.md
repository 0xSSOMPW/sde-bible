# Q: How do CPU and memory limits work in Docker, and what is exit code 137?

**Answer:**

Container resource limits are enforced by **cgroups**, not Docker itself. Misunderstanding them is the most common cause of mysterious container kills, "noisy neighbors," and CPU throttling that doesn't show in `top`.

### Memory Limits

```bash
docker run --memory=512m --memory-swap=512m myapp
```

- `--memory` sets the *hard* cap. Exceed it → kernel OOM-killer fires inside the container's cgroup.
- `--memory-swap` is the *total* of RAM + swap. Set it equal to `--memory` to disable swap.
- `--memory-reservation` is a *soft* limit; only enforced under host memory pressure.

When the cap is hit:

```
container process tries malloc → kernel sees cgroup memory.max exceeded
                              → OOM killer scores cgroup processes
                              → SIGKILL the largest
                              → exit code 137 (128 + 9)
                              → Docker shows "OOMKilled: true"
```

Diagnose:

```bash
docker inspect mycontainer | grep -i oom
dmesg | grep -i "killed process"
```

> [!NOTE]
> The OOM kill is **synchronous** and **uncatchable**. Your app gets no chance to flush, drain, or page someone. Plan capacity so OOMs are an anomaly, not a tuning strategy.

### CPU Limits — the Two Flavors

Two independent knobs, often confused:

**1. CPU shares (relative weight).**

```bash
docker run --cpu-shares=512 myapp   # default is 1024
```

Only matters under contention. If the host is idle, a 512-share container can still use 100% CPU. Useful for prioritization, not for hard caps.

**2. CPU quota (absolute cap, "CFS bandwidth control").**

```bash
docker run --cpus=1.5 myapp
# Equivalent to:
# --cpu-period=100000 --cpu-quota=150000
```

Hard cap. The container's cgroup gets 150 ms of CPU per 100 ms wall-clock window. Once exhausted, **all threads in the cgroup are throttled** until the next window.

### The CFS Throttling Trap

A container limited to 1 CPU running a 16-thread Java app:

- Each request needs 200 ms CPU spread across threads.
- All 16 threads run in parallel, burn the 100 ms quota in ~7 ms.
- Throttled for 93 ms doing nothing.
- p99 latency tanks even though average CPU usage looks low (~7%!).

Diagnose:

```bash
cat /sys/fs/cgroup/cpu.stat
# nr_throttled    <-- non-zero is suspicious
# throttled_usec  <-- time spent throttled
```

Fixes:
- Reduce thread count to match `--cpus` (or set runtime flags: `-XX:ActiveProcessorCount`, `GOMAXPROCS`).
- Increase `--cpu-period` to give larger windows (rarely needed).
- Raise the limit. CFS throttling is brutal on bursty workloads.

### Memory: JVM/Runtime Awareness

A common failure mode: `--memory=2g` with a JVM that defaults heap to "25% of physical RAM" — JVM sees the host's 64 GB, sets heap to 16 GB, container OOMs immediately on warm-up.

Modern runtimes (Java 11+, Node, Go) detect cgroup limits *if* you use `cgroups v2` and don't disable container support. Defensive flags:

```bash
# JVM
-XX:MaxRAMPercentage=75.0 -XX:+UseContainerSupport

# Node
NODE_OPTIONS="--max-old-space-size=1536"   # in MB, leave room for off-heap

# Go
GOMEMLIMIT=1500MiB   # Go 1.19+
```

### What the Numbers Actually Mean

```
Host: 8 cores, 16 GB RAM

docker run --cpus=2 --memory=4g myapp
   ↓
cgroup writes:
  cpu.max:    200000 100000   (2 cores worth per 100 ms)
  memory.max: 4294967296

Inside container:
  nproc                 → 8     (sees host CPUs, not the limit!)
  cat /proc/meminfo     → host total RAM
```

The container *cannot tell* it's limited via the usual files — that's why runtimes must read `/sys/fs/cgroup/...` directly. Tools that don't are why you see ancient bugs about JVM choosing wrong heap sizes.

### Decision Table

| Symptom | Likely cause | Fix |
|---------|-------------|-----|
| Exit code 137, `OOMKilled: true` | Hard memory cap exceeded | Raise `--memory` or fix leak |
| Exit code 137, `OOMKilled: false` | Host OOM killed *something*, took your container with it | Raise host capacity; set memory limits everywhere |
| Container slow, CPU usage low | CFS throttling | Match thread/runtime concurrency to `--cpus` |
| Container slow during bursts only | Quota period too short | Same fix, or raise quota |
| `--cpu-shares` not enforcing | Only fires under contention; expected | Use `--cpus` for hard caps |

### Interview Follow-ups

- *"What's the difference between requests and limits in Kubernetes?"* — Requests = scheduler hint and `cpu.weight`; limits = `cpu.max` / `memory.max`. Memory request != memory cap. Limit > request can produce surprises during contention.
- *"Why is OOMKilled different from a regular crash?"* — Sent by the kernel based on cgroup accounting; no signal handler will catch it.
- *"Should you set memory limit = request?"* — In K8s, yes if you want `Guaranteed` QoS class. Reduces preemption risk.
