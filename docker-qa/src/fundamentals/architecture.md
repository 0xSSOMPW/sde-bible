# Q: Explain the Docker Architecture.

**Answer:**

Docker uses a **client-server architecture** with three main components:

### 1. Docker Client (`docker` CLI)
The command-line interface that you interact with. When you run a command like `docker run`, the client sends it as an API request to the Docker daemon. The client can communicate with the daemon locally (via a Unix socket) or remotely (via TCP).

### 2. Docker Daemon (`dockerd`)
The background service (server) that does all the heavy lifting. It manages:
- Building images
- Running containers
- Pulling/pushing images from registries
- Managing networks and volumes

The daemon exposes a REST API that the client talks to.

### 3. Docker Registry (e.g., Docker Hub)
A storage and distribution system for Docker images. When you `docker pull nginx`, the daemon fetches the image from Docker Hub (the default public registry). Companies also run **private registries** (e.g., AWS ECR, GCR, Harbor).

### How They Work Together

```
┌──────────────┐       REST API       ┌──────────────────┐
│ Docker Client │ ──────────────────▶  │  Docker Daemon   │
│  (docker CLI) │                      │   (dockerd)      │
└──────────────┘                      │                  │
                                       │  ┌────────────┐ │
                                       │  │ Containers  │ │
                                       │  ├────────────┤ │
                                       │  │   Images    │ │
                                       │  ├────────────┤ │
                                       │  │  Volumes    │ │
                                       │  ├────────────┤ │
                                       │  │  Networks   │ │
                                       │  └────────────┘ │
                                       └────────┬─────────┘
                                                │
                                       ┌────────▼─────────┐
                                       │  Docker Registry  │
                                       │  (Docker Hub,     │
                                       │   ECR, GCR, etc.) │
                                       └──────────────────┘
```

### Under the Hood: containerd & runc
The Docker daemon doesn't actually run containers directly. It delegates to:
1. **containerd**: A high-level container runtime that manages the full container lifecycle (image transfer, storage, execution).
2. **runc**: A low-level OCI-compliant runtime that actually creates and runs containers using Linux kernel features (namespaces, cgroups).

> [!TIP]
> When discussing Docker architecture in interviews, mentioning **containerd** and **runc** shows a deeper understanding. Kubernetes, for example, talks directly to containerd (not Docker) since Docker was deprecated as a Kubernetes runtime in v1.24.
