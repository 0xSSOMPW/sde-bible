# Q: What is the PID 1 problem in Docker, and how do you handle SIGTERM correctly?

**Answer:**

In a container, your process runs as **PID 1** — the same special status as `init` on a normal Linux system. PID 1 carries kernel-enforced responsibilities most programs do not handle, and the result is **signals ignored** and **zombie processes accumulating**. This is the "PID 1 problem."

### The Two Specific Issues

**1. Signal handling is opt-in for PID 1.**

The kernel does *not* deliver signals to PID 1 unless that process has explicitly installed a handler. For every other PID, the default action runs (e.g., SIGTERM → terminate). This means:

```bash
docker run --rm myapp
^C
# ...waits for stop timeout (10s default), then SIGKILL
```

`docker stop` sends SIGTERM. If your binary doesn't handle it (and you ran it as PID 1), nothing happens. Docker waits, then sends SIGKILL.

**2. Zombie reaping.**

When a child process exits, it stays as a zombie until its parent calls `wait()`. Normally `init` reaps orphans. In a container, *your* process is init — and if you never call `wait()` on grandchildren, zombies accumulate until the PID table fills.

### When You Hit This

- Shell-form `CMD`: `CMD node server.js` — `node` becomes PID 1.
  - Older Node didn't install a SIGTERM handler. Container won't stop gracefully.
- Wrapper scripts: `CMD ["./start.sh"]` and `start.sh` does `exec node server.js`.
  - `exec` replaces the shell, so node *does* become PID 1 — but inherits whatever signal mask the shell set up.
- Forking servers without proper reaping.

### Fix 1: Use `exec` Form and Handle Signals

```dockerfile
# Bad — shell form: actually runs `/bin/sh -c "node server.js"`
# sh becomes PID 1 and doesn't forward SIGTERM.
CMD node server.js

# Good — exec form: node is PID 1 directly.
CMD ["node", "server.js"]
```

And in the app:

```javascript
process.on('SIGTERM', () => {
  server.close(() => process.exit(0));
});
```

### Fix 2: Add an Init Process

Docker ships an option:

```bash
docker run --init myapp
```

That injects `tini` as PID 1. `tini` forwards signals to your child and reaps zombies. Equivalent in Compose:

```yaml
services:
  app:
    image: myapp
    init: true
```

Or bake it in:

```dockerfile
RUN apk add --no-cache tini
ENTRYPOINT ["/sbin/tini", "--"]
CMD ["node", "server.js"]
```

Kubernetes has no `--init` flag; either bake tini in or write a real PID 1.

### Graceful Shutdown in Practice

```
docker stop / k8s rolling update
        │
        ▼
   SIGTERM → PID 1
        │
        ▼  (your handler runs)
   ┌────────────────────────┐
   │ 1. Stop accepting new  │
   │    connections         │
   │ 2. Drain in-flight     │
   │    requests            │
   │ 3. Flush metrics/logs  │
   │ 4. Close DB pools      │
   │ 5. exit(0)             │
   └────────────────────────┘
        │
        ▼
   After grace period (10s default) → SIGKILL
```

Configure the grace period:

```bash
docker stop --time=30 mycontainer
```

```yaml
# Kubernetes
terminationGracePeriodSeconds: 30
```

### Common Mistakes

| Mistake | Result |
|---------|--------|
| `CMD npm start` | npm forks node, doesn't forward SIGTERM. 10s stalls on every stop. |
| Catching SIGTERM but not closing keep-alive connections | Pod stuck in `Terminating` for full grace period |
| Long-lived requests vs short grace period | Mid-flight requests get SIGKILLed |
| `preStop` hook for cleanup, but app exits immediately on SIGTERM | Cleanup races shutdown |

### Detecting Zombies

Inside container:

```bash
ps -ef | awk '$8 == "Z" { print }'
# Or
cat /proc/PID/status | grep State
# State: Z (zombie)
```

If you see them, your PID 1 isn't reaping. Add `--init` or tini.

> [!NOTE]
> Kubernetes sends SIGTERM, waits `terminationGracePeriodSeconds`, then SIGKILL. The endpoints controller removes the pod from the Service *concurrently* with sending SIGTERM, not before — so you may receive new connections for a brief window after SIGTERM. The standard fix is a `preStop` sleep (3–5s) before your app starts draining.

### Interview Follow-ups

- *"What exit code does SIGKILL produce?"* — 137 (128 + 9). SIGTERM-caught-and-clean-exit = 0; uncaught SIGTERM = 143.
- *"Why does `docker stop` wait 10 seconds?"* — Default grace period, configurable via `--time` or `STOPSIGNAL`/`STOPTIMEOUT`.
- *"Difference between `--init` and tini in the image?"* — Functionally similar. `--init` is convenient but runtime-dependent (some K8s setups don't enable it). Baking tini guarantees behavior.
