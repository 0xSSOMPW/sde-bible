# Q: Distroless vs Alpine vs slim — which base image, and why?

**Answer:**

Choice of base image is the single biggest lever for image size, attack surface, and build complexity. The defaults (`ubuntu`, `python`, `node`) are convenient but huge. Smaller bases pay back in pull times, CVE counts, and supply-chain auditability.

### The Spectrum

```
ubuntu:24.04        ~ 78 MB      full distro, package manager, shell, libc
debian:bookworm     ~ 124 MB     similar
python:3.12         ~ 1 GB       Debian + Python + dev tools
python:3.12-slim    ~ 130 MB     Debian + Python, no dev tools
alpine:3.20         ~ 7 MB       musl libc, busybox, apk
distroless/static   ~ 2 MB       libc only (for static binaries)
distroless/cc       ~ 17 MB      glibc + libssl
distroless/java21   ~ 230 MB     JRE only
scratch             0 bytes      empty — bring your own everything
```

### What "Distroless" Actually Is

Google's distroless images contain **only what your application needs to run**:
- A libc (or static linkage and no libc).
- A few support libraries (`libssl`, `zoneinfo`, CA certs).
- **No shell. No package manager. No coreutils. No `cat`, `ls`, `bash`.**

Implication: you can't `docker exec ... sh`. The lack of shell is a feature — many CVEs require an attacker to chain a shell after exploitation.

### Multi-Stage Build with Distroless

Go binary:

```dockerfile
FROM golang:1.22 AS build
WORKDIR /src
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -o /app ./cmd/server

FROM gcr.io/distroless/static-debian12:nonroot
COPY --from=build /app /app
USER nonroot:nonroot
ENTRYPOINT ["/app"]
```

Final image: ~12 MB total. No CVEs in the base. No shell.

Java 21:

```dockerfile
FROM eclipse-temurin:21-jdk AS build
WORKDIR /src
COPY . .
RUN ./mvnw -DskipTests package && jar xf target/app.jar

FROM gcr.io/distroless/java21-debian12:nonroot
COPY --from=build /src/BOOT-INF/lib /app/lib
COPY --from=build /src/BOOT-INF/classes /app/classes
COPY --from=build /src/META-INF /app/META-INF
USER nonroot
ENTRYPOINT ["java", "-cp", "/app:/app/lib/*", "org.springframework.boot.loader.launch.JarLauncher"]
```

### Distroless vs Alpine

| Aspect | Alpine | Distroless |
|--------|--------|-----------|
| Size | ~7 MB base, larger with deps | 2–25 MB |
| libc | musl | glibc (most flavors) |
| Package manager | `apk` | none |
| Shell | `/bin/sh` (busybox) | none |
| DNS quirks | musl's resolver differs from glibc | matches glibc |
| CGO / native libs | musl rebuild often needed | works as-is |
| Image rebuild for security update | rebuild + `apk upgrade` | rebuild from upstream distroless |

The **musl trap**: Python wheels, Node native modules, JIT runtimes are usually published for glibc. On Alpine they either fail or fall back to slow pure-Python/JS. If you use Alpine and your build has native deps, expect to compile from source.

### Scratch (Empty Image)

The smallest possible base. No libc, no zoneinfo, no CA certs. Works for fully static binaries:

```dockerfile
FROM scratch
COPY app /app
COPY ca-certificates.crt /etc/ssl/certs/
ENTRYPOINT ["/app"]
```

Caveats:
- TLS: you must copy `/etc/ssl/certs/ca-certificates.crt`.
- Timezones: must copy `/usr/share/zoneinfo`.
- DNS: musl/glibc-static handle this; pure CGO_DISABLED Go is fine.
- No `/tmp`: create it explicitly if your app needs it.

### Comparing CVE Surface

Snyk/Trivy scans on a Go web service:

```
ubuntu:24.04           120 known CVEs in base packages
debian:bookworm-slim    35 CVEs
alpine:3.20              4 CVEs
distroless/static        0 CVEs
```

CVE-free isn't the same as bug-free, but a smaller base means fewer paths an attacker can exploit if they get RCE.

### Debugging Without a Shell

The pain point: you can't `kubectl exec -it pod -- sh` into a distroless container.

Options:
1. **Build a `:debug` variant.** Distroless ships `gcr.io/distroless/static-debian12:debug` with busybox + `sh`.
2. **Ephemeral debug containers** (Kubernetes 1.25+):
   ```bash
   kubectl debug -it mypod --image=busybox --target=mycontainer
   ```
   Attaches a debug sidecar sharing process namespace.
3. **Live debugging from a different container** in the same pod, sharing `processNamespaceShareProcessNamespace: true`.

### Choosing for Your Runtime

| Runtime | Recommended base |
|---------|------------------|
| Go (static, CGO_ENABLED=0) | `distroless/static:nonroot` or `scratch` |
| Rust (musl static) | `scratch` or `alpine` |
| Rust (glibc) | `distroless/cc` |
| Python | `python:3.12-slim` (distroless Python lacks pip) |
| Node | `node:20-alpine` or `gcr.io/distroless/nodejs20-debian12` |
| Java | `eclipse-temurin:21-jre-alpine` or `gcr.io/distroless/java21` |
| Anything FFI-heavy | `debian:slim` over Alpine |

### Common Mistakes

| Mistake | Fix |
|---------|-----|
| Using full `python:3.12` in production | `-slim` minimum; better, distroless |
| Alpine for a Python app with native wheels | Use `python:3.12-slim` (Debian glibc) |
| Distroless + dynamic linking (CGO without static flag) | Either link statically or use `distroless/cc` |
| Single-stage build leaving compiler in image | Multi-stage; final stage from minimal base |
| Forgetting CA certs in `scratch` | Copy `/etc/ssl/certs/ca-certificates.crt` |

> [!NOTE]
> Smaller images are not only faster to pull — they're cheaper to scan, faster to load into Kubernetes, and reduce the blast radius of any kernel exploit. A 1 GB image is a regression you should justify.

### Interview Follow-ups

- *"Why does Alpine make Python apps slow sometimes?"* — musl's malloc and DNS resolver differ; native wheels (numpy, cryptography) fall back to pure-Python implementations.
- *"How does distroless get security updates?"* — Google rebuilds the base regularly; you rebuild your image to pick up new digests. No in-image `apt upgrade`.
- *"Can you sign distroless images?"* — Yes, with `cosign`. Google publishes signed manifests.
