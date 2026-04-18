# Summary

- [Introduction](./README.md)

---

# Fundamentals

- [Core Concepts]()
  - [Containers vs Virtual Machines](./fundamentals/containers-vs-vms.md)
  - [Docker Architecture](./fundamentals/architecture.md)
  - [Docker Image vs Container](./fundamentals/image-vs-container.md)

---

# Images & Dockerfiles

- [Dockerfile]()
  - [CMD vs ENTRYPOINT](./images/cmd-vs-entrypoint.md)
  - [COPY vs ADD](./images/copy-vs-add.md)
  - [Multi-Stage Builds](./images/multi-stage-builds.md)
  - [Layer Caching & Optimization](./images/layer-caching.md)

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
- [Production Patterns]()
  - [Health Checks](./production/health-checks.md)
  - [Logging Best Practices](./production/logging.md)
  - [Docker vs Kubernetes](./production/docker-vs-k8s.md)
