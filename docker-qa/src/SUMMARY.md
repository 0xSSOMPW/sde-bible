# Summary

- [Introduction](./README.md)

---

# Fundamentals

- [Core Concepts]()
  - [Containers vs Virtual Machines](./fundamentals/containers-vs-vms.md)
  - [Docker Architecture](./fundamentals/architecture.md)
  - [Docker Image vs Container](./fundamentals/image-vs-container.md)
  - [Namespaces & cgroups (How Containers Work)](./fundamentals/namespaces-cgroups.md)

---

# Images & Dockerfiles

- [Dockerfile]()
  - [CMD vs ENTRYPOINT](./images/cmd-vs-entrypoint.md)
  - [COPY vs ADD](./images/copy-vs-add.md)
  - [Multi-Stage Builds](./images/multi-stage-builds.md)
  - [Layer Caching & Optimization](./images/layer-caching.md)
  - [BuildKit & buildx](./images/buildkit.md)
  - [Tags vs Digests (Pinning)](./images/tags-vs-digests.md)
  - [Build Context & .dockerignore](./images/dockerignore-build-context.md)
  - [Multi-Arch Builds (amd64 + arm64)](./images/multi-arch-builds.md)

---

# Container Lifecycle

- [Containers]()
  - [Container States & Lifecycle](./containers/lifecycle.md)
  - [exec vs attach](./containers/exec-vs-attach.md)
  - [Restart Policies](./containers/restart-policies.md)

---

# Networking

- [Docker Networking]()
  - [Network Types (Bridge, Host, Overlay)](./networking/network-types.md)
  - [Port Mapping & Expose](./networking/port-mapping.md)
  - [Container DNS & Service Discovery](./networking/dns-service-discovery.md)

---

# Storage

- [Docker Storage]()
  - [Volumes vs Bind Mounts vs tmpfs](./storage/volumes-vs-bind-mounts.md)
  - [Data Persistence Strategies](./storage/data-persistence.md)
  - [Storage Drivers (overlay2, CoW)](./storage/storage-drivers.md)

---

# Docker Compose

- [Compose]()
  - [What is Docker Compose?](./compose/what-is-compose.md)
  - [depends_on vs healthcheck](./compose/depends-on-vs-healthcheck.md)
  - [Environment Variables & Secrets](./compose/env-and-secrets.md)

---

# Security & Production

- [Security]()
  - [Running as Non-Root](./security/non-root.md)
  - [Image Scanning & Minimizing Attack Surface](./security/image-scanning.md)
  - [Distroless vs Alpine vs slim](./security/distroless-minimal-images.md)
  - [Rootless Docker & Non-root Containers](./security/rootless-docker.md)
- [Production Patterns]()
  - [Health Checks](./production/health-checks.md)
  - [Logging Best Practices](./production/logging.md)
  - [Docker vs Kubernetes](./production/docker-vs-k8s.md)
  - [Resource Limits & Exit Code 137](./production/resource-limits.md)
  - [PID 1 Problem & Signal Handling](./production/pid1-signal-handling.md)
  - [Health Checks Deep Dive (Startup, Readiness, Liveness)](./production/health-checks-deep.md)
