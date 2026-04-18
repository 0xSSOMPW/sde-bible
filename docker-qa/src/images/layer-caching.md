# Q: How does Docker Layer Caching work? How do you optimize a Dockerfile?

**Answer:**

Understanding layer caching is essential for building images fast and keeping them small.

### How Layers Work
Every instruction in a Dockerfile (`FROM`, `RUN`, `COPY`, `ADD`, `ENV`, etc.) creates a new **layer**. Layers are stacked, read-only, and cached. When you rebuild an image, Docker checks each instruction:
- If the instruction and its inputs **haven't changed**, Docker reuses the cached layer (instant).
- If anything **has changed**, Docker invalidates that layer **and all layers after it** (the cache "busts").

### The Cache Busting Problem

```dockerfile
# ❌ Bad order: Cache busts on EVERY code change
FROM node:20-alpine
WORKDIR /app
COPY . .                    # Any code change invalidates THIS layer
RUN npm install             # ...which forces this to re-run (slow!)
CMD ["node", "index.js"]
```

Every time you change a single line of code, `COPY . .` changes, which invalidates the cache for `npm install`. You end up reinstalling all dependencies from scratch on every build.

### The Fix: Order by Change Frequency

```dockerfile
# ✅ Good order: Dependencies cached separately from code
FROM node:20-alpine
WORKDIR /app
COPY package.json package-lock.json ./   # Changes rarely
RUN npm ci                               # Cached unless package.json changes
COPY . .                                 # Code changes only bust THIS layer
CMD ["node", "index.js"]
```

Now, if you only change application code, Docker reuses the cached `npm ci` layer and only re-runs `COPY . .` — saving minutes on every build.

### Optimization Best Practices

**1. Use `.dockerignore`**
Just like `.gitignore`, a `.dockerignore` file prevents unnecessary files from being sent to the Docker build context.
```
node_modules
.git
*.md
.env
dist
```

**2. Combine `RUN` commands**
Each `RUN` instruction creates a new layer. Combine related commands to reduce layer count and image size.
```dockerfile
# ❌ Bad: 3 layers (including cached apt lists)
RUN apt-get update
RUN apt-get install -y curl
RUN rm -rf /var/lib/apt/lists/*

# ✅ Good: 1 layer, cleanup in same step
RUN apt-get update && \
    apt-get install -y --no-install-recommends curl && \
    rm -rf /var/lib/apt/lists/*
```

**3. Use specific base image tags**
```dockerfile
# ❌ Bad: `latest` changes unpredictably, breaks cache
FROM node:latest

# ✅ Good: Pinned version, reproducible
FROM node:20.11-alpine3.18
```

**4. Use multi-stage builds** (see the previous chapter).

**5. Prefer Alpine or Distroless base images**
- `node:20` → ~900MB
- `node:20-alpine` → ~130MB  
- `node:20-slim` → ~200MB
