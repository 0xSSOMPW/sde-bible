# Q: Why should containers run as a non-root user?

**Answer:**

By default, the process inside a Docker container runs as **root (UID 0)**. This is a significant security risk because if an attacker breaks out of the container (a container escape), they gain root access to the host machine.

### The Risk
```dockerfile
# ❌ Default: runs as root
FROM node:20-alpine
WORKDIR /app
COPY . .
CMD ["node", "index.js"]
# Inside the container: whoami → root
```

If someone exploits a vulnerability in your Node.js app, they have root-level access inside the container. Combined with a kernel exploit, this could mean root on the host.

### The Fix: Create and Use a Non-Root User

```dockerfile
FROM node:20-alpine
WORKDIR /app

# Create a non-root user and group
RUN addgroup -S appgroup && adduser -S appuser -G appgroup

# Install deps as root (needs permissions)
COPY package*.json ./
RUN npm ci --only=production

# Copy app code
COPY . .

# Change ownership of app files to the non-root user
RUN chown -R appuser:appgroup /app

# Switch to non-root user for all subsequent commands
USER appuser

EXPOSE 3000
CMD ["node", "index.js"]
# Inside the container: whoami → appuser
```

### Other Approaches

**1. Use `--user` at runtime:**
```bash
docker run --user 1000:1000 myapp
```

**2. Use official base images that already set a non-root user:**
Many official images (like `node`) include a pre-created user:
```dockerfile
FROM node:20-alpine
USER node  # Built-in non-root user
```

**3. Read-only filesystem:**
```bash
docker run --read-only --tmpfs /tmp myapp
```
This prevents any writes to the container filesystem, further limiting attack surface.

### Additional Hardening

```bash
# Drop all Linux capabilities, add back only what's needed
docker run --cap-drop ALL --cap-add NET_BIND_SERVICE myapp

# Prevent privilege escalation
docker run --security-opt no-new-privileges myapp
```

> [!TIP]
> In Kubernetes, you enforce this via `securityContext` in the Pod spec:
> ```yaml
> securityContext:
>   runAsNonRoot: true
>   runAsUser: 1000
>   readOnlyRootFilesystem: true
> ```
