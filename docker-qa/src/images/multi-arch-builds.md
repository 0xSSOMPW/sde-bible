# Q: How do you build multi-arch (amd64 + arm64) Docker images correctly?

**Answer:**

Apple Silicon, AWS Graviton, and Raspberry Pi all run **arm64**. CI runners and most cloud servers run **amd64**. Shipping one image that works everywhere means building a **multi-platform manifest**: a single tag that points to per-architecture image variants.

### The Manifest List

```
me/app:1.0
  ├── linux/amd64   →  sha256:aaaa...
  ├── linux/arm64   →  sha256:bbbb...
  └── linux/arm/v7  →  sha256:cccc...
```

When a daemon pulls `me/app:1.0`, the registry returns the manifest list. The daemon picks the variant matching its platform.

### Building with `docker buildx`

```bash
# Create a builder that supports multiple platforms
docker buildx create --name multi --use --bootstrap

# Build and push
docker buildx build \
  --platform linux/amd64,linux/arm64 \
  -t me/app:1.0 \
  --push .
```

`--push` is required for multi-platform — local Docker can hold only one platform at a time. The manifest list is assembled in the registry.

### How It Builds Non-Native

The builder runs the Dockerfile *for each platform*. Three strategies:

**1. QEMU emulation** (default, simplest).

```bash
docker run --privileged --rm tonistiigi/binfmt --install all
```

Registers QEMU as the binfmt handler for non-native ELFs. Now `docker buildx` can "run" an arm64 binary on an amd64 host. **Slow** — 3–10× native.

**2. Native cross-compilation** (fast, requires support in the language).

```dockerfile
FROM --platform=$BUILDPLATFORM golang:1.22 AS build
ARG TARGETOS TARGETARCH
RUN CGO_ENABLED=0 GOOS=$TARGETOS GOARCH=$TARGETARCH \
    go build -o /app ./cmd/server

FROM --platform=$TARGETPLATFORM gcr.io/distroless/static
COPY --from=build /app /app
ENTRYPOINT ["/app"]
```

Key tricks:
- `--platform=$BUILDPLATFORM` on the build stage → runs natively (fast).
- Use `GOOS`/`GOARCH` to cross-compile to the target.
- `--platform=$TARGETPLATFORM` on the final stage → copies into the right arch base.

This is **5–10× faster** than QEMU for Go, Rust (with cross), and C/C++ where cross-compilers exist.

**3. Native builders** (one builder per platform, federated).

```bash
docker buildx create --name multi --node arm-node --platform linux/arm64 \
  ssh://user@arm-runner
docker buildx create --append --name multi --node amd-node --platform linux/amd64
docker buildx use multi
```

Each platform builds on its own native host. Fastest, but requires extra infrastructure.

### Common Variables

```
$TARGETPLATFORM     linux/arm64
$TARGETOS           linux
$TARGETARCH         arm64
$TARGETVARIANT      v8         (for arm)
$BUILDPLATFORM      linux/amd64  (host doing the build)
$BUILDOS            linux
$BUILDARCH          amd64
```

### Dockerfile Patterns

**Pure interpreter (Python, Ruby, Node)**:

```dockerfile
FROM python:3.12-slim
COPY requirements.txt .
RUN pip install -r requirements.txt        # pip resolves arch from wheels
COPY . .
CMD ["python", "app.py"]
```

Most wheels publish both arm64 and amd64 since Python 3.10+. If a wheel doesn't, pip falls back to source build — slow under QEMU.

**Go (cross-compile, fastest)**:

```dockerfile
FROM --platform=$BUILDPLATFORM golang:1.22 AS build
ARG TARGETOS TARGETARCH
WORKDIR /src
COPY . .
RUN CGO_ENABLED=0 GOOS=$TARGETOS GOARCH=$TARGETARCH go build -o /app

FROM scratch
COPY --from=build /app /app
ENTRYPOINT ["/app"]
```

**Rust (cross-compile via `cross`)**:

```dockerfile
FROM --platform=$BUILDPLATFORM rust:1 AS build
ARG TARGETARCH
RUN case "$TARGETARCH" in \
      "arm64") rustup target add aarch64-unknown-linux-musl;; \
      "amd64") rustup target add x86_64-unknown-linux-musl;; \
    esac
COPY . .
RUN cargo build --release --target $(rustc -vV | sed -n 's|host: ||p')
```

In practice, use the `cross` crate to avoid handwriting all this.

**Java**: bytecode is platform-independent, but the base JRE isn't. Pick a multi-platform base:

```dockerfile
FROM eclipse-temurin:21-jre
COPY target/app.jar /app.jar
ENTRYPOINT ["java", "-jar", "/app.jar"]
```

`eclipse-temurin` publishes both architectures — `buildx` selects each.

### Output Without Pushing

```bash
docker buildx build --platform linux/amd64,linux/arm64 \
  -t me/app --output type=oci,dest=app.tar .
```

Useful for air-gapped environments. `docker buildx imagetools` can later load it.

### Verifying

```bash
docker buildx imagetools inspect me/app:1.0
# Lists each platform's digest, OS, architecture

# Pull only one:
docker pull --platform linux/arm64 me/app:1.0
```

### CI: GitHub Actions Example

```yaml
- uses: docker/setup-qemu-action@v3
- uses: docker/setup-buildx-action@v3
- uses: docker/login-action@v3
  with:
    registry: ghcr.io
    username: ${{ github.actor }}
    password: ${{ secrets.GITHUB_TOKEN }}
- uses: docker/build-push-action@v6
  with:
    context: .
    platforms: linux/amd64,linux/arm64
    push: true
    tags: ghcr.io/me/app:${{ github.sha }}
    cache-from: type=gha
    cache-to:   type=gha,mode=max
```

Note: native arm64 runners (`runs-on: ubuntu-24.04-arm`) are now available — pair with matrix builds + `buildx imagetools create` to assemble a manifest list without QEMU.

### Native Matrix Build (Fastest)

```yaml
strategy:
  matrix:
    include:
      - runner: ubuntu-24.04
        platform: linux/amd64
      - runner: ubuntu-24.04-arm
        platform: linux/arm64

# Each job builds and pushes a per-arch digest tag
# Final job assembles manifest list:

- run: |
    docker buildx imagetools create \
      -t ghcr.io/me/app:${{ github.sha }} \
      ghcr.io/me/app:${{ github.sha }}-amd64 \
      ghcr.io/me/app:${{ github.sha }}-arm64
```

Native builds are 5–10× faster than QEMU; this pattern is the production CI standard now.

### Common Mistakes

| Mistake | Fix |
|---------|-----|
| Building one arch, pushing, deploying to arm64 cluster | Empty manifest list = "no matching manifest" error |
| QEMU build everything, 30-min CI | Use cross-compile or native runners |
| `RUN` step uses arch-specific download URL | Use `$TARGETARCH` to pick the URL |
| Forgetting `--push` (or `--load` for single-platform) | Multi-platform images can't sit in local cache |
| Stale binfmt registration after host upgrade | Re-run `tonistiigi/binfmt --install all` |

> [!NOTE]
> Cross-compile when your language supports it. QEMU is a fine fallback but never an optimization. Match the build pattern to the language; don't put one Dockerfile through QEMU when `GOOS=...` is one env var away.

### Interview Follow-ups

- *"Why was the manifest list invented?"* — Tag = single SHA was a constraint. Manifest list (a.k.a. OCI Image Index) lets one tag refer to a fan-out of per-arch images.
- *"Does this affect image size?"* — No — a client only pulls its platform's variant. Total registry storage is sum of all platforms.
- *"What's `--load`?"* — Single-platform alternative to `--push`: loads the built image into local Docker. Multi-platform `--load` is not supported because local Docker can't hold a manifest list.
