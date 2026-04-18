# Q: What are the different Container States and the Container Lifecycle?

**Answer:**

A Docker container goes through several states during its lifecycle. Understanding these is important for debugging and orchestration.

### Container States

```
docker create     docker start     (process exits)     docker rm
    │                  │                  │                │
    ▼                  ▼                  ▼                ▼
 CREATED ──────▶ RUNNING ──────▶ EXITED ──────▶ REMOVED
                    │    ▲
          docker    │    │  docker
          pause     │    │  unpause
                    ▼    │
                  PAUSED
```

1. **Created**: Container exists but hasn't started yet (`docker create`).
2. **Running**: The container's main process (PID 1) is actively executing (`docker start` or `docker run`).
3. **Paused**: The container's processes are frozen (using `cgroup freezer`). Memory state is preserved (`docker pause`).
4. **Exited**: The main process has finished or crashed. The writable layer is still on disk.
5. **Removed**: The container and its writable layer are deleted (`docker rm`).

### Key Commands

```bash
# Create + Start in one command
docker run -d --name myapp nginx

# View running containers
docker ps

# View ALL containers (including stopped ones)
docker ps -a

# Stop gracefully (sends SIGTERM, then SIGKILL after grace period)
docker stop myapp

# Stop immediately (sends SIGKILL)
docker kill myapp

# Remove a stopped container
docker rm myapp

# Force remove a running container
docker rm -f myapp

# Remove ALL stopped containers
docker container prune
```

### The PID 1 Problem
The container's main process is always PID 1. When PID 1 exits, the **entire container stops**, regardless of whether other processes are still running inside it.

> [!IMPORTANT]
> This is why `docker run` should always run the main application process as the foreground command (not as a background daemon). If your entrypoint script runs the app with `&` (background) and then exits, the container will immediately stop.
