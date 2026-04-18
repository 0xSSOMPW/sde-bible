# Q: What is the difference between Volumes, Bind Mounts, and tmpfs?

**Answer:**

Docker provides three mechanisms for persisting data or sharing files between the host and containers.

### 1. Volumes (Managed by Docker)
Volumes are the **preferred mechanism** for persisting data. Docker fully manages them — they're stored in a dedicated directory on the host (`/var/lib/docker/volumes/`) and are completely abstracted from the host filesystem.

```bash
# Create a named volume
docker volume create mydata

# Use it
docker run -v mydata:/app/data myapp
# or (more explicit long syntax):
docker run --mount type=volume,source=mydata,target=/app/data myapp
```

### 2. Bind Mounts (Host Path → Container Path)
A bind mount maps a **specific file or directory on the host** directly into the container. The host and container see the exact same files in real-time.

```bash
# Mount current directory into the container
docker run -v $(pwd):/app myapp
# or:
docker run --mount type=bind,source=$(pwd),target=/app myapp
```

### 3. tmpfs Mounts (In-Memory Only)
Data is stored in the host's **RAM** only. It is never written to disk and is lost when the container stops. Useful for sensitive data that should not persist.

```bash
docker run --tmpfs /app/secrets myapp
# or:
docker run --mount type=tmpfs,target=/app/secrets myapp
```

### Comparison

| Feature | Volume | Bind Mount | tmpfs |
|---|---|---|---|
| **Stored on** | Docker-managed area on disk | Any host path | Host RAM |
| **Managed by** | Docker CLI (`docker volume`) | You (host filesystem) | Kernel |
| **Portable** | Yes (works across environments) | No (depends on host path) | No |
| **Performance** | Excellent | Excellent | Fastest (RAM) |
| **Persists after stop** | ✅ Yes | ✅ Yes (on host) | ❌ No |
| **Pre-populated** | ✅ Yes (from image) | ❌ No (overwrites) | ❌ No |
| **Use case** | Database storage, app data | Dev hot-reload, config files | Secrets, temp caches |

### When to Use What?
- **Volumes**: Production data (databases, uploads). Portable and manageable.
- **Bind Mounts**: Local development (mount source code for hot-reload).
- **tmpfs**: Storing sensitive info (tokens, keys) that should never hit disk.

> [!IMPORTANT]
> Bind mounts can be dangerous in production because they give containers direct access to the host filesystem. A container running as root with a bind mount to `/` could access or modify any file on the host.
