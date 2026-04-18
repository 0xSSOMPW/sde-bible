# Q: What is the difference between `CMD` and `ENTRYPOINT` in a Dockerfile?

**Answer:**

Both `CMD` and `ENTRYPOINT` define what command runs when a container starts, but they behave very differently when users pass arguments at runtime.

### `CMD` — The Default Command (Easily Overridden)
`CMD` sets the default command and/or arguments for the container. However, **it is completely replaced** if the user provides a command when running the container.

```dockerfile
FROM ubuntu
CMD ["echo", "Hello from CMD"]
```
```bash
docker run myimage
# Output: Hello from CMD

docker run myimage echo "I replaced CMD"
# Output: I replaced CMD  (CMD was completely overridden)
```

### `ENTRYPOINT` — The Fixed Executable (Not Easily Overridden)
`ENTRYPOINT` sets the main executable for the container. **User-provided arguments are appended** to the entrypoint, not used to replace it.

```dockerfile
FROM ubuntu
ENTRYPOINT ["echo", "Hello from"]
```
```bash
docker run myimage
# Output: Hello from

docker run myimage "Docker World"
# Output: Hello from Docker World  (argument was appended)
```

### The Power Combo: `ENTRYPOINT` + `CMD`
The most common production pattern is using them together. `ENTRYPOINT` defines the fixed executable, and `CMD` provides default arguments that can be overridden.

```dockerfile
FROM python:3.11-slim
ENTRYPOINT ["python"]
CMD ["app.py"]
```
```bash
docker run myimage
# Runs: python app.py (default)

docker run myimage test.py
# Runs: python test.py (CMD overridden, ENTRYPOINT kept)
```

### Summary

| Feature | `CMD` | `ENTRYPOINT` |
|---|---|---|
| **Purpose** | Default command/args | Fixed executable |
| **Override behavior** | Completely replaced by `docker run` args | Args are *appended* to it |
| **Best for** | Default arguments | The main process |

> [!TIP]
> To override `ENTRYPOINT` at runtime, you must explicitly use the `--entrypoint` flag:
> `docker run --entrypoint /bin/bash myimage`
