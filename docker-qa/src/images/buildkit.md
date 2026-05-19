# Q: What is BuildKit, and how does it differ from the legacy Docker builder?

**Answer:**

BuildKit is the modern Docker image builder, default since Docker 23.0. It replaces the old "classic" builder with a parallel, content-addressable, plugin-friendly engine. Skipping its features is the difference between a 20-minute and a 2-minute CI build.

### Why It Exists

The legacy builder:
- Processed a Dockerfile **strictly top to bottom, one stage at a time**.
- Could not run independent stages in parallel.
- Cache was layer-hash–based and easy to invalidate by accident.
- No way to mount build-time secrets without baking them into layers.
- No native cross-platform builds.

BuildKit fixes all of these.

### What BuildKit Adds

| Feature | Syntax | Why |
|---------|--------|-----|
| Parallel stages | (automatic) | Multi-stage with independent stages builds them concurrently |
| Cache mounts | `RUN --mount=type=cache,target=/root/.npm npm ci` | Persist package caches across builds without bloating image |
| Secret mounts | `RUN --mount=type=secret,id=npmrc,target=/root/.npmrc npm ci` | Use credentials at build time, never written to a layer |
| SSH mounts | `RUN --mount=type=ssh git clone git@...` | Clone private repos without baking keys |
| Bind mounts (from context) | `RUN --mount=type=bind,source=.,target=/src ...` | Avoid `COPY` overhead for scratch use |
| Distributed cache | `--cache-to type=registry,ref=...` | Share cache across CI runners via a registry |
| Multi-platform | `docker buildx build --platform linux/amd64,linux/arm64` | Single command, multi-arch manifest |

### Enabling It

Default in Docker 23+. Otherwise:

```bash
export DOCKER_BUILDKIT=1
# Or use buildx (the BuildKit CLI front-end)
docker buildx build .
```

For multi-platform or remote builds, create a builder:

```bash
docker buildx create --name multi --use --bootstrap
docker buildx build --platform linux/amd64,linux/arm64 -t me/app:1.0 --push .
```

### Cache Mounts: The Highest-ROI Feature

Without cache mount, a small `package.json` change re-downloads every npm dep:

```dockerfile
COPY package*.json ./
RUN npm ci          # ~90s on a cold cache
COPY . .
RUN npm run build
```

With:

```dockerfile
# syntax=docker/dockerfile:1.7
COPY package*.json ./
RUN --mount=type=cache,target=/root/.npm \
    npm ci          # ~5s warm: tarballs reused
COPY . .
RUN --mount=type=cache,target=/app/.next/cache \
    npm run build
```

Note the **first line**: `# syntax=docker/dockerfile:1.7` opts into newer Dockerfile features.

### Secret Mounts

Pre-BuildKit pattern (insecure — secret ends up in image history):

```dockerfile
ARG NPM_TOKEN
RUN echo "//registry.npmjs.org/:_authToken=$NPM_TOKEN" > .npmrc && npm ci
```

Even with multi-stage cleanup, the layer's history records the ARG. BuildKit way:

```bash
docker build --secret id=npmrc,src=$HOME/.npmrc .
```

```dockerfile
# syntax=docker/dockerfile:1.7
RUN --mount=type=secret,id=npmrc,target=/root/.npmrc \
    npm ci
```

The file exists only during the `RUN`. Never persisted in any layer.

### Distributed Cache for CI

Local layer cache is gone every time a CI runner is freshly provisioned. Push cache to a registry:

```bash
docker buildx build \
  --cache-from type=registry,ref=ghcr.io/me/app:buildcache \
  --cache-to   type=registry,ref=ghcr.io/me/app:buildcache,mode=max \
  -t ghcr.io/me/app:$SHA --push .
```

`mode=max` exports cache for every layer (not just final stages). Bigger cache image, faster subsequent builds.

### Multi-Platform Builds

```bash
docker buildx build \
  --platform linux/amd64,linux/arm64 \
  -t me/app:1.0 --push .
```

BuildKit emulates non-native architectures via QEMU. Native cross-compilation is faster:

```dockerfile
FROM --platform=$BUILDPLATFORM golang:1.22 AS build
ARG TARGETOS TARGETARCH
RUN CGO_ENABLED=0 GOOS=$TARGETOS GOARCH=$TARGETARCH go build -o /app
```

`$BUILDPLATFORM` = the builder's arch (fast native build); `$TARGETPLATFORM` = the image's arch.

### BuildKit vs Legacy Comparison

| Aspect | Legacy | BuildKit |
|--------|--------|----------|
| Parallel stages | No | Yes |
| Build cache | Per-layer, in-image | Content-addressable, mountable |
| Secrets | Via ARG (leaks) | Native mount type |
| Multi-platform | No | Yes (buildx) |
| Dockerfile features | Frozen | Per-`syntax=` directive — versioned |
| Output formats | Only image | Image, OCI tarball, local files, registry direct |

### Interview Follow-ups

- *"Why is my `COPY . .` invalidating cache on every build?"* — Because the context changed. Use `.dockerignore` to exclude irrelevant files; use `--mount=type=bind` for transient access.
- *"What is `docker buildx`?"* — The Docker CLI plugin that drives BuildKit. Default builder in newer Docker.
- *"How do I see what BuildKit is actually doing?"* — `--progress=plain` instead of the default TTY view; pair with `BUILDKIT_PROGRESS=plain`.
