# Q: What are the different Docker Restart Policies?

**Answer:**

Restart policies control whether a container is automatically restarted when it exits or when the Docker daemon restarts.

### Available Policies

```bash
docker run --restart <policy> myimage
```

| Policy | Behavior |
|---|---|
| `no` | Never restart the container (default). |
| `on-failure[:max-retries]` | Restart only if the container exits with a **non-zero exit code**. Optionally limit the number of retries. |
| `always` | Always restart the container, regardless of exit code. Also restarts when the Docker daemon starts. |
| `unless-stopped` | Same as `always`, but does **not** restart if the container was manually stopped before the daemon restart. |

### Examples

```bash
# Restart up to 5 times on failure
docker run --restart on-failure:5 myapp

# Always keep the container running (survives daemon restarts)
docker run --restart always nginx

# Same as always, but respects manual stops
docker run --restart unless-stopped nginx
```

### `always` vs `unless-stopped`
The subtle but critical difference:

1. You run a container with `--restart always`.
2. You manually `docker stop` it.
3. The Docker daemon restarts (e.g., server reboot).
4. **Result: The container starts again** (because the policy is `always`).

With `unless-stopped`:
1. You run a container with `--restart unless-stopped`.
2. You manually `docker stop` it.
3. The Docker daemon restarts.
4. **Result: The container stays stopped** (it respects your manual stop).

> [!TIP]
> For production services, prefer `unless-stopped`. It auto-recovers from crashes while still respecting your intent when you explicitly stop a container for maintenance.

### In Docker Compose

```yaml
services:
  web:
    image: nginx
    restart: unless-stopped
```
