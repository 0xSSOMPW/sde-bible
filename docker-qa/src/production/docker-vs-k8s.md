# Q: When would you use Docker (Compose/Swarm) vs Kubernetes?

**Answer:**

This is a high-level architecture question that interviewers use to gauge your understanding of container orchestration.

### Docker Compose
- **Scope**: Single host only.
- **Use case**: Local development, CI/CD test environments, small single-server deployments.
- **Complexity**: Minimal. A single YAML file.
- **Scaling**: `docker compose up --scale web=3` (basic, no load balancer).
- **Networking**: Automatic service discovery on the same host.

### Docker Swarm
- **Scope**: Multi-host cluster (built into Docker Engine).
- **Use case**: Simple production setups, small teams that want orchestration without Kubernetes complexity.
- **Features**: Service discovery, load balancing, rolling updates, secrets management.
- **Scaling**: `docker service scale web=10` (across multiple nodes).
- **Learning curve**: Low (if you know Docker, you know 80% of Swarm).

### Kubernetes (K8s)
- **Scope**: Multi-host cluster (industry standard for container orchestration).
- **Use case**: Large-scale production, microservices, multi-team environments.
- **Features**: Everything Swarm has, plus: auto-scaling (HPA/VPA), self-healing, RBAC, custom resource definitions (CRDs), Ingress controllers, service mesh support, advanced scheduling.
- **Scaling**: Handles thousands of nodes and hundreds of thousands of pods.
- **Learning curve**: Steep. Requires understanding of Pods, Deployments, Services, ConfigMaps, etc.

### Comparison Table

| Feature | Compose | Swarm | Kubernetes |
|---|---|---|---|
| **Multi-host** | ❌ | ✅ | ✅ |
| **Auto-scaling** | ❌ | ❌ | ✅ (HPA) |
| **Self-healing** | ❌ | ✅ (basic) | ✅ (advanced) |
| **Rolling updates** | ❌ | ✅ | ✅ |
| **Load balancing** | ❌ | ✅ (built-in) | ✅ (Service + Ingress) |
| **Secrets** | File-based | ✅ (encrypted) | ✅ (encrypted) |
| **Community/Ecosystem** | N/A | Declining | Dominant |
| **Setup complexity** | Minutes | Hours | Days |

### When to Use What?
- **Compose**: You're developing locally or running a small app on a single server.
- **Swarm**: You need multi-host orchestration but want something simpler than Kubernetes. (Note: Swarm adoption is declining; most teams go straight to K8s.)
- **Kubernetes**: You need production-grade orchestration, auto-scaling, advanced networking, or you're operating at scale.

> [!NOTE]
> In interviews, it's perfectly acceptable to say: "We used Docker Compose for local dev and Kubernetes for production." This shows practical understanding of using the right tool for the right environment.
