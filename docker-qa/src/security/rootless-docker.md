# Q: Non-root container vs rootless Docker — what's the difference?

**Answer:**

These are two **independent** security controls people often confuse:

- **Non-root container**: the *process inside* the container runs as a non-zero UID.
- **Rootless Docker**: the *Docker daemon itself* runs as a non-root user.

You can run either, both, or neither. They protect against different threats.

### Non-Root Container

```dockerfile
FROM node:20-alpine
RUN addgroup -S app && adduser -S app -G app
WORKDIR /app
COPY --chown=app:app . .
USER app
CMD ["node", "server.js"]
```

Inside the container, the process is `app`, UID 1000. Not `root`. So even if your app is exploited:
- Can't bind ports < 1024 (unless granted `CAP_NET_BIND_SERVICE`).
- Can't `chmod` files it doesn't own.
- Can't modify `/etc/passwd`, install packages, etc.

Threat it mitigates: **in-container privilege misuse**. Doesn't protect against kernel exploits — if you escape the container as `app`, the host still maps that UID somewhere.

### Rootless Docker

The Docker daemon (`dockerd`) and `containerd` themselves run as a non-root user using **user namespaces**. The host sees `dockerd` as `alice`, but the daemon manages namespaces that look like UID 0 to processes inside containers.

Install / enable:

```bash
dockerd-rootless-setuptool.sh install
systemctl --user start docker
export DOCKER_HOST=unix://$XDG_RUNTIME_DIR/docker.sock
```

Now `docker run alpine id` shows `uid=0(root)` *inside* the container — but on the host, that process is actually running as `alice` mapped through `/etc/subuid` and `/etc/subgid`.

Threat it mitigates: **container escape → root on host**. Even with a kernel CVE, an attacker only gets `alice`-level privileges.

### How They Combine

| Daemon | Container UID | Result |
|--------|--------------|--------|
| Root | root (UID 0) | Worst: any escape = host root |
| Root | non-root | App-level mistakes contained; escape = host root |
| Rootless | root | Container "root" maps to your user on host |
| Rootless | non-root | Defense in depth — recommended |

### `--user` and `USER`

```bash
docker run --user 1000:1000 myimage          # ad-hoc
```

```dockerfile
USER 1000:1000          # numeric — works without /etc/passwd entry
```

Numeric form is safer for distroless images that lack `/etc/passwd`. Kubernetes preference:

```yaml
securityContext:
  runAsNonRoot: true
  runAsUser: 1000
  runAsGroup: 1000
  fsGroup: 1000             # files in mounts owned by this group
```

`runAsNonRoot: true` makes the kubelet **refuse to start** a container whose image declares `USER 0` or doesn't declare a USER. Cheap insurance.

### Port Binding Below 1024

A non-root container can't bind `:80`/`:443`. Three fixes:

1. **Bind to a high port and map**: container listens on `:8080`, host publishes `80:8080`. Most common.
2. **Grant the capability**: `docker run --cap-add NET_BIND_SERVICE` (or `setcap 'cap_net_bind_service=+ep'` on the binary).
3. **Sysctl on Linux** lowers the privileged port range to 0 (host-wide; not always advisable).

### File Permissions in Volumes

A common headache:

```bash
docker run --user 1000 -v $(pwd):/app myimage
# Inside, files owned by host UID 1000 — works if host user is 1000.
```

If host UID doesn't match container UID, writes fail with `EACCES`. Two fixes:
- Match UIDs (`--user $(id -u):$(id -g)`).
- `--volume` named volume + entrypoint that `chown`s on first start.

Rootless Docker bypasses this because user namespace mapping handles it transparently.

### Capabilities

Linux capabilities split root's privileges into ~40 buckets. Docker drops most by default:

```
Default kept:  CHOWN  DAC_OVERRIDE  FSETID  FOWNER  MKNOD
               NET_RAW  SETGID  SETUID  SETFCAP  SETPCAP
               NET_BIND_SERVICE  SYS_CHROOT  KILL  AUDIT_WRITE
Default dropped: SYS_ADMIN, SYS_PTRACE, NET_ADMIN, ...
```

Tighten further:

```bash
docker run --cap-drop ALL --cap-add NET_BIND_SERVICE myimage
```

In K8s:

```yaml
securityContext:
  capabilities:
    drop: ["ALL"]
    add: ["NET_BIND_SERVICE"]
```

### Other Hardening Flags

```bash
docker run \
  --read-only \                        # rootfs read-only; mount tmpfs for writable paths
  --tmpfs /tmp:rw,size=64m \
  --security-opt no-new-privileges \   # blocks setuid escalation
  --pids-limit 200 \                   # fork-bomb mitigation
  --memory 512m --cpus 1 \             # resource caps
  myimage
```

`no-new-privileges` is particularly cheap and high-impact — it disables `setuid` binary escalation inside the container.

### Rootless Limitations

- **No `--net=host`** (no privileged port use without `setcap` on the daemon binary).
- **No overlayfs by default on some kernels** — falls back to `fuse-overlayfs` (slower).
- **Cgroups v2 required** for proper resource limits.
- **Docker Compose works**, but `docker swarm` is limited.

### Threat Model Comparison

| Threat | Non-root container | Rootless daemon | Both |
|--------|-------------------|-----------------|------|
| App RCE → file tamper inside container | Partial (only files non-root owns) | No help | Same |
| App RCE → spawn local process | Limited capabilities | Limited host blast | Best |
| Container escape via kernel CVE | Host root | Host user only | Host user only |
| Daemon compromise | Host root | Host user only | Host user only |
| Image with malicious entrypoint | Runs as image's USER | Same | Same |

> [!NOTE]
> The two settings stack. If you're picking only one for a production deployment, **non-root container** delivers more bang per setup minute. Add **rootless** when you can — it's the bigger blast-radius reducer.

### Interview Follow-ups

- *"What is `setuid` and how does `no-new-privileges` help?"* — `setuid` binaries gain their owner's privileges on exec. `no-new-privileges` makes the kernel ignore that — closing a common escalation path post-RCE.
- *"How does K8s `runAsNonRoot` differ from `runAsUser: 1000`?"* — `runAsNonRoot` rejects images that *would* run as 0. `runAsUser` overrides the image's USER directive.
- *"User namespaces gotchas in production?"* — File ownership in shared volumes is the most common headache; the UID inside doesn't match outside. K8s `fsGroup` smooths this for emptyDir/CSI, not hostPath.
