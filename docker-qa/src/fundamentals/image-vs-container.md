# Q: What is the difference between a Docker Image and a Container?

**Answer:**

This is a deceptively simple but critical distinction:

### Docker Image
An image is a **read-only template** that contains the application code, runtime, libraries, environment variables, and configuration files needed to run an application. Think of it as a **class** in OOP or a **blueprint** for a house.

Images are built in **layers**. Each instruction in a Dockerfile (e.g., `RUN`, `COPY`, `ADD`) creates a new layer. Layers are stacked on top of each other and are cached, which makes rebuilds extremely fast.

### Docker Container
A container is a **running instance** of an image. Think of it as an **object** instantiated from a class, or a house built from a blueprint. You can create multiple containers from the same image.

When a container starts, Docker adds a thin **writable layer** on top of the read-only image layers. All file changes (new files, modifications, deletions) happen in this writable layer.

### Analogy

```
Image  = Class definition (immutable blueprint)
Container = Object/Instance (running process with mutable state)

One Image → Many Containers (just like one Class → many Objects)
```

### Key Differences

| Feature | Image | Container |
|---|---|---|
| **State** | Immutable (read-only) | Mutable (has a writable layer) |
| **Stored as** | Layers on disk | Running process + writable layer |
| **Created by** | `docker build` or `docker pull` | `docker run` or `docker create` |
| **Persistence** | Persists until explicitly deleted | Ephemeral by default (data lost on removal) |
| **Sharing** | Pushed to registries | Cannot be pushed (must be `docker commit`'d into an image first) |

### Common Follow-Up: What is `docker commit`?
You can take a running container's writable layer and freeze it into a new image:
```bash
docker commit <container_id> my-custom-image:v1
```

> [!CAUTION]
> Using `docker commit` in production is considered bad practice. Always use a `Dockerfile` for reproducible, version-controlled image builds.
