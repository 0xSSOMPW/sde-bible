# Q: What are Linux namespaces and cgroups, and how do they make containers?

**Answer:**

Containers are not a single Linux feature — they are an *application* of two unrelated kernel mechanisms: **namespaces** (isolation) and **cgroups** (resource limits). Docker is mostly glue around these.

### The Two Halves

| Mechanism | Provides | Without it |
|-----------|----------|-----------|
| **Namespaces** | Each container sees its own PIDs, network, mounts, users, etc. | All processes share one global view |
| **cgroups** (control groups) | CPU, memory, I/O limits per group of processes | One container can starve the host |

A container = a process tree wrapped in a set of namespaces + placed in a set of cgroups.

### The Namespaces

```
┌────────────────────────────────────────────────────────────┐
│  Namespace  │  Isolates                                    │
├────────────────────────────────────────────────────────────┤
│  pid        │  Process IDs (container sees its own PID 1)  │
│  net        │  Network interfaces, routes, iptables, ports │
│  mnt        │  Filesystem mounts (chroot on steroids)      │
│  uts        │  hostname, domainname                        │
│  ipc        │  SysV IPC, POSIX message queues              │
│  user       │  UID/GID mappings (root in container ≠ root) │
│  cgroup     │  cgroup root view                            │
│  time       │  CLOCK_MONOTONIC offset (newer)              │
└────────────────────────────────────────────────────────────┘
```

You can poke at them on any Linux box:

```bash
ls -l /proc/$$/ns/
# lrwxrwxrwx 1 user user 0 ... net -> 'net:[4026531992]'
# lrwxrwxrwx 1 user user 0 ... pid -> 'pid:[4026531836]'
```

Two processes with the same inode number after the colon share that namespace.

Inside a container, `ps aux` shows only the container's processes — because the `pid` namespace doesn't expose the host's processes at all. From the *host*, you can see them by inode:

```bash
sudo lsns -t pid
```

### cgroups

cgroups v2 (unified hierarchy, default on modern distros) lets you cap resources per group:

```bash
# Memory cap example for a Docker container
docker run --memory=512m --cpus=1.5 nginx
```

Translates to writes under `/sys/fs/cgroup/...`:

```
memory.max = 536870912
cpu.max    = 150000 100000     # 1.5 cores
```

If a container exceeds `memory.max`, the kernel OOM-kills a process **inside the container's cgroup** — typically PID 1 — and the container exits with **137** (128 + SIGKILL 9).

### Putting It Together: How `docker run` Works

```
1. docker CLI -> dockerd -> containerd -> runc
2. runc reads OCI spec (config.json), then:
   a. clone(CLONE_NEW{PID,NET,MNT,UTS,IPC,USER,CGROUP})  <- create namespaces
   b. mount the image rootfs (overlay2), pivot_root      <- new filesystem view
   c. write cgroup limits to /sys/fs/cgroup/...           <- enforce resources
   d. apply seccomp + AppArmor/SELinux profiles           <- syscall filtering
   e. drop capabilities                                   <- least privilege
   f. exec the container's ENTRYPOINT                     <- run user process
```

Note: there is no "container" kernel object. After step (f), it's just a Linux process — distinguished only by which namespaces and cgroups it belongs to.

### Why This Matters for Interviews

- *"Why is the container so much lighter than a VM?"* — No guest kernel, no hypervisor. A container is a normal process; the kernel is shared.
- *"What does `--privileged` actually do?"* — Disables most isolation: gives all capabilities, doesn't drop user namespace, mounts `/dev`, removes the default AppArmor/seccomp profiles. Dangerous.
- *"How does the container's PID 1 get its responsibilities?"* — PID namespace makes the entrypoint PID 1, which means it inherits zombie reaping and signal handling duties. (See: tini, PID 1 problem.)
- *"How do containers on the same host talk to each other?"* — Their `net` namespaces are connected via a virtual bridge (`docker0`) and pairs of `veth` devices.

### Common Misconceptions

| Belief | Reality |
|--------|---------|
| "Containers are mini-VMs" | They share the host kernel; no isolation boundary as strong as a VM |
| "Docker invented containers" | Namespaces existed before Docker (LXC, Solaris zones, FreeBSD jails); Docker made the UX usable |
| "Containers are secure by default" | They're a *better* default than running everything as root, but a kernel exploit escapes the namespace boundary |

> [!NOTE]
> A useful mental model: **a container is just a process with strange glasses on.** Namespaces are the glasses; cgroups are the diet.
