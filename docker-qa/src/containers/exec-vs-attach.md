# Q: What is the difference between `docker exec` and `docker attach`?

**Answer:**

Both commands let you interact with a running container, but they connect to very different things.

### `docker attach`
Attaches your terminal's stdin/stdout/stderr to the container's **main process (PID 1)**. You are essentially watching and interacting with the same process that `docker run` started.

```bash
docker run -d --name myapp python app.py
docker attach myapp
# You are now connected to the stdout of `python app.py`
```

**Danger:** If you press `Ctrl+C` while attached, it sends `SIGINT` to PID 1, which **stops the container** entirely.

> [!WARNING]
> Use `Ctrl+P` then `Ctrl+Q` to detach from a container without killing it. This is the "detach sequence."

### `docker exec`
Starts a **brand new, separate process** inside the running container. The new process runs alongside PID 1 without affecting it.

```bash
docker run -d --name myapp nginx
docker exec -it myapp /bin/bash
# Opens a new bash shell inside the container
# Exiting this shell does NOT stop the container
```

### Key Differences

| Feature | `docker attach` | `docker exec` |
|---|---|---|
| **Connects to** | PID 1 (main process) | A new, separate process |
| **Use case** | Viewing main process output | Debugging, running ad-hoc commands |
| **Ctrl+C** | Stops the container | Only kills the exec'd process |
| **Multiple terminals** | All see the same PID 1 output | Each gets an independent process |

### When to Use Which?
- **`docker exec`** (99% of the time): Debugging, inspecting files, running one-off commands, opening a shell.
- **`docker attach`**: Rare. Useful when you need to interact with the stdin of an interactive main process (e.g., a REPL).
