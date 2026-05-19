# Q: How does Docker layered storage actually work (overlay2, CoW)?

**Answer:**

Container images are stacks of read-only filesystem layers, unioned with a thin writable layer at runtime. The mechanism is **copy-on-write (CoW)** via a kernel storage driver — almost always `overlay2` on modern Linux.

### The Stack

```
        Container's view:
        ┌──────────────────────┐
        │  /  (unified view)   │
        └──────────────────────┘
                  ▲
                  │ overlay union
                  │
   ┌──────────────────────────┐    ←  upperdir  (writable, CoW)
   │  layer 4 (your changes)  │
   ├──────────────────────────┤    \
   │  layer 3 (COPY . .)      │     \
   ├──────────────────────────┤      ├  lowerdirs  (read-only)
   │  layer 2 (RUN npm ci)    │     /   ordered top → bottom
   ├──────────────────────────┤    /
   │  layer 1 (FROM node:20)  │
   └──────────────────────────┘
```

Each `FROM`/`RUN`/`COPY`/`ADD` in a Dockerfile creates one read-only layer. When the container starts, Docker mounts an `overlay2` filesystem with these as `lowerdir`s and a fresh empty directory as `upperdir`.

### Copy-on-Write

A read sees the file from whichever layer has it last (top wins). A write triggers CoW:

1. Find the file in the lowerdirs.
2. **Copy** it up to the upperdir.
3. Mutate the copy.

```
echo "hi" >> /etc/config
   │
   ▼
overlay2 sees /etc/config in lowerdir
   │
   ▼
Copies entire file to upperdir, appends
```

CoW is cheap for small files, expensive for big ones (a `dd if=/dev/zero of=/var/lib/big bs=1M count=1024` writes 1 GB to the upperdir even if the original was 1 GB).

### Image Layer = Layer Tarball

```
/var/lib/docker/
├── image/overlay2/
│   ├── layerdb/sha256/<digest>/    <- metadata, diff IDs, parent chain
│   └── ...
├── overlay2/
│   ├── <layer_id>/diff/             <- actual files for this layer
│   ├── <layer_id>/link              <- short symlink name
│   ├── <layer_id>/lower             <- lower chain (multi-line)
│   └── <container_id>/merged/       <- live overlay mount point
```

Each layer's `diff/` directory is a normal filesystem subtree containing only the files **changed in that layer** (additions and modifications) plus **whiteout files** (`.wh.<name>`) marking deletions.

### Why Layer Order Matters

Cache invalidation is layer-by-layer, top-down. Change a layer, all layers above rebuild:

```dockerfile
FROM node:20-alpine
WORKDIR /app
COPY . .                     # ❌ any source change invalidates the layer above
RUN npm ci
```

vs.

```dockerfile
FROM node:20-alpine
WORKDIR /app
COPY package*.json ./
RUN npm ci                   # cached unless package.json changes
COPY . .                     # invalidates only this layer + below
```

Order from **least likely to change** at the top to **most likely** at the bottom.

### Layer Size and Squash

A layer's size = bytes physically added by that step. `RUN apt-get install foo && apt-get clean` does **not** make the layer smaller — the layer still contains everything written before the clean (Docker can't see the temporal "clean up").

```dockerfile
# ❌ 200 MB layer even after clean
RUN apt-get update && apt-get install -y curl
RUN apt-get clean && rm -rf /var/lib/apt/lists/*

# ✅ Same step, single layer, no orphan files
RUN apt-get update \
 && apt-get install -y curl \
 && apt-get clean \
 && rm -rf /var/lib/apt/lists/*
```

Both `apt-get update` and the install must be in the same `RUN` because the cache from `update` is gigantic.

`docker build --squash` (experimental) collapses all your image's layers into one — saves bytes but kills layer cache for downstream builds.

### Storage Drivers

| Driver | Linux requirement | Status |
|--------|------------------|--------|
| `overlay2` | kernel ≥ 4.0 | Default; recommended |
| `fuse-overlayfs` | userspace overlay | Rootless Docker |
| `btrfs` | btrfs filesystem | Niche; snapshots free |
| `zfs` | zfs filesystem | Niche; snapshots free |
| `devicemapper` | block-level | Deprecated |
| `aufs` | aufs-patched kernel | Legacy, mostly Debian/Ubuntu old |
| `vfs` | none | No CoW, slow, last-resort |

Check yours:

```bash
docker info | grep "Storage Driver"
# Storage Driver: overlay2
```

### `RUN` That Writes Big Files Is Bad Practice

A 2 GB tarball downloaded, extracted, deleted — all in one `RUN` — still costs 2 GB+ during the build (in the intermediate filesystem) and may produce a layer that includes the temporary file if the `rm` doesn't happen in the same step. With BuildKit, use cache mounts:

```dockerfile
# syntax=docker/dockerfile:1.7
RUN --mount=type=cache,target=/tmp/dl \
    curl -o /tmp/dl/foo.tar.gz https://... \
 && tar xf /tmp/dl/foo.tar.gz -C /opt
```

Cache mount lives outside any layer.

### Examining Layers

```bash
docker history myimage:tag --no-trunc
# Shows each layer's command and size

docker save myimage:tag | tar -xv -C /tmp/extract
# Image tarball contains layer tarballs; useful for forensics

# Or use dive (third-party):
dive myimage:tag    # interactive layer explorer
```

### Performance Implications

| Pattern | Cost |
|---------|------|
| Reading a file from a deep layer | Negligible (file lookup walks chain) |
| First write to a file | CoW copy (file-sized I/O burst) |
| Writing many small files | Each is a small CoW; usually fine |
| Writing to `/proc`, `/sys` | Not in overlay; kernel-managed |
| Database directories in the container FS | Slow + ephemeral — use a volume |

> [!NOTE]
> Rule: data that changes lives on a **volume**, not in the container filesystem. The CoW model is optimized for read-mostly workloads; write-heavy paths (DB files, log files, build caches) bypass it via mounts.

### Common Mistakes

| Mistake | Fix |
|---------|-----|
| One `RUN` per `apt-get` install line | Combine into one to avoid intermediate layer bloat |
| `RUN curl ... | RUN tar ... | RUN rm ...` | Single chained `RUN` so the cleanup is in the same layer |
| Writing logs/DB files inside container FS | Use `-v` volume |
| Believing `--squash` is a free win | It breaks the cache for everyone downstream |
| Counting `du -sh` inside container | Reports usage of the whole overlay; not layer sizes |

### Interview Follow-ups

- *"What's a whiteout file?"* — A special marker file overlay uses to represent deletion in a higher layer. Looks like `.wh.<filename>`.
- *"Why is `overlay2` better than `aufs`?"* — Mainline kernel support, simpler design, page-cache sharing between containers using the same lowerdirs.
- *"Why is `vfs` so slow?"* — No CoW. Each new layer is a full copy of the previous. Used only when overlay isn't available (some CI sandboxes).
