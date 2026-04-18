# Q: What are the different Docker Network Types?

**Answer:**

Docker provides several built-in network drivers. Understanding them is crucial for designing multi-container applications.

### 1. Bridge Network (Default)
The default network type for standalone containers. Docker creates a virtual bridge (`docker0`) on the host and assigns each container a private IP address within that bridge's subnet.

```bash
# Containers on the default bridge can communicate via IP, 
# but NOT by container name (no automatic DNS).
docker run -d --name app1 nginx
docker run -d --name app2 nginx
# app2 cannot reach app1 via http://app1 (only by IP)

# Custom bridge networks DO support DNS resolution:
docker network create mynet
docker run -d --name app1 --network mynet nginx
docker run -d --name app2 --network mynet nginx
# Now app2 CAN reach app1 via http://app1 ✅
```

> [!IMPORTANT]
> Always use **custom bridge networks** instead of the default bridge. Custom bridges provide automatic DNS resolution, better isolation, and the ability to connect/disconnect containers dynamically.

### 2. Host Network
Removes network isolation entirely. The container shares the host's network stack directly. No port mapping is needed — the container's ports are the host's ports.

```bash
docker run --network host nginx
# nginx is now accessible on the host's port 80 directly
```

**Pros:** Best network performance (no NAT overhead).
**Cons:** Port conflicts if multiple containers use the same port. Not available on Docker Desktop (macOS/Windows).

### 3. Overlay Network
Enables communication between containers running on **different Docker hosts** (across machines). Used in Docker Swarm and Kubernetes environments.

```bash
docker network create -d overlay my-overlay
```

Uses VXLAN tunneling under the hood to encapsulate container traffic across physical network boundaries.

### 4. None Network
Completely disables networking for the container. The container only has a loopback interface.

```bash
docker run --network none myapp
# No external network access at all
```

**Use case:** Security-sensitive batch processing where no network communication should be possible.

### Summary

| Driver | Scope | DNS | Use Case |
|---|---|---|---|
| `bridge` | Single host | Custom only | Default for standalone containers |
| `host` | Single host | N/A | Max performance, no isolation needed |
| `overlay` | Multi-host | Yes | Swarm/K8s clusters |
| `none` | N/A | N/A | Security, isolated batch jobs |
