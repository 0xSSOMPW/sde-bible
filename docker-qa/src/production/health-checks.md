# Q: How do Health Checks work in Docker?

**Answer:**

A health check is a command that Docker runs periodically inside a container to determine if the application is healthy and functioning correctly.

### Defining Health Checks

**In a Dockerfile:**
```dockerfile
FROM node:20-alpine
WORKDIR /app
COPY . .

HEALTHCHECK --interval=30s --timeout=5s --start-period=10s --retries=3 \
    CMD curl -f http://localhost:3000/health || exit 1

CMD ["node", "index.js"]
```

**In Docker Compose:**
```yaml
services:
  api:
    build: .
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3000/health"]
      interval: 30s
      timeout: 5s
      start_period: 10s
      retries: 3
```

### Health Check Parameters

| Parameter | Default | Description |
|---|---|---|
| `interval` | 30s | Time between health checks |
| `timeout` | 30s | Max time to wait for the check to complete |
| `start_period` | 0s | Grace period for the container to initialize |
| `retries` | 3 | Number of consecutive failures before marking `unhealthy` |

### Health Check States

| State | Meaning |
|---|---|
| `starting` | Container just launched, within `start_period` |
| `healthy` | Health check command exited with code 0 |
| `unhealthy` | Health check failed `retries` times consecutively |

```bash
# Check container health
docker inspect --format='{{.State.Health.Status}}' mycontainer
# Output: healthy

docker ps
# CONTAINER ID   STATUS
# abc123         Up 5 min (healthy)
```

### Common Health Check Commands

```dockerfile
# HTTP endpoint check (requires curl in the image)
HEALTHCHECK CMD curl -f http://localhost:3000/health || exit 1

# TCP port check (no curl needed)
HEALTHCHECK CMD nc -z localhost 3000 || exit 1

# PostgreSQL readiness
HEALTHCHECK CMD pg_isready -U postgres || exit 1

# Redis ping
HEALTHCHECK CMD redis-cli ping || exit 1
```

> [!TIP]
> For Alpine-based images that don't have `curl`, use `wget`:
> ```dockerfile
> HEALTHCHECK CMD wget --spider -q http://localhost:3000/health || exit 1
> ```
> Or for images without any HTTP tools, use a simple Node.js script or a compiled binary health checker.
