# Q: What is the build context, and why does `.dockerignore` matter so much?

**Answer:**

The **build context** is the directory tree sent from your client to the Docker daemon when you run `docker build`. Misunderstanding it is the #1 cause of slow builds, accidental secrets in images, and "why did my cache invalidate?" mysteries.

### What Gets Sent

```bash
docker build -t app .
                    ^ this dot is the build context
```

Docker tars up **every file** under `.` (minus `.dockerignore` matches) and streams it to the daemon — even files you never `COPY`. The daemon can only see files in this context; it cannot reach back to your filesystem.

```
Client                          Daemon
  │   build context: 480 MB       │
  │ ────────────────────────────► │  starts evaluating Dockerfile
  │                               │  COPY package.json → finds it in context
```

A 5 GB context (often: `node_modules`, `target/`, `.git`, dataset files) wastes:
- Network/IPC: every build re-sends the tarball.
- Cache: a single file change in the context can invalidate `COPY` layers.
- Disk: BuildKit caches the context.

### `.dockerignore`

Same syntax as `.gitignore`. Lives next to the Dockerfile (or wherever the context root is).

```
# Build artifacts
node_modules
dist
target
build
*.pyc
__pycache__

# Dev files
.git
.gitignore
.github
.vscode
.idea
*.md
README*

# Secrets — most important
.env
.env.*
*.pem
*.key
id_rsa*

# Tests/coverage
coverage
test-results
.pytest_cache

# OS
.DS_Store
Thumbs.db

# Allow-list pattern: ignore everything, then include
*
!src/
!package.json
!package-lock.json
```

Verify what gets sent:

```bash
docker build --progress=plain .
# Look for "transferring context" size

# Or:
tar czf /tmp/ctx.tgz --exclude-from .dockerignore .
ls -lh /tmp/ctx.tgz
```

### `COPY` and Cache Invalidation

```dockerfile
COPY package.json package-lock.json ./
RUN npm ci
COPY . .
RUN npm run build
```

Layer cache for `COPY package.json` is invalidated **only** if either file changed. Layer for `COPY . .` is invalidated if *any* file in the context changed — including `README.md`, `tests/`, `.git`. Without `.dockerignore` filtering, you re-run `npm run build` whenever any commit touches the repo.

### The Secrets Trap

`.env`, `.npmrc` with auth tokens, SSH keys — if they're in the context and someone `COPY`s them, they're in the image **forever** as a layer. Even removing them in a later layer leaves them in the history.

```dockerfile
COPY . .                     # ← copies .env if not ignored
RUN rm .env                  # ← .env still exists in the previous layer
```

`.dockerignore` is the first line of defense. The second is BuildKit secret mounts (`--mount=type=secret`), which never become part of any layer.

### Allow-list Pattern

Default-deny is safer than default-allow. Instead of listing what to ignore, list what to include:

```
# Ignore everything
*

# Then explicitly include
!src/
!public/
!package.json
!package-lock.json
!tsconfig.json
!Dockerfile
```

Adding a new file to the repo? You must opt it in. Less convenient, but no surprises.

### Negation Subtlety

```
node_modules
!node_modules/some-package
```

This doesn't work like `.gitignore` for directory contents. Docker stops at the parent — once `node_modules/` is excluded, individual files inside can't be re-included. You'd have to re-include the directory and then exclude what you don't want.

### Multi-stage Builds and Context

The context is shared across all stages. The `FROM ... AS build` stage and the final stage see the same context. There's no "per-stage" filtering — `.dockerignore` is global.

### Remote Context

You can build from a URL or git repo:

```bash
docker build https://github.com/me/app.git#main
docker build https://github.com/me/app.git#main:subdir
```

The repo is cloned server-side; `.dockerignore` still applies.

### Context-less Build (`-f -`)

```bash
echo "FROM scratch" | docker build -
```

No context at all — no `COPY` possible. Useful for trivial images.

### Common Mistakes

| Mistake | Fix |
|---------|-----|
| `.git` in context (10s of MB on big repos) | Add to `.dockerignore` |
| `node_modules` in context, then `RUN npm ci` reinstalls it anyway | Always ignore — wastes network + cache |
| Secrets in context, removed in later RUN | Use `--mount=type=secret` or never put them in context |
| `.dockerignore` not in the same directory as `-f` Dockerfile in newer Docker | Use `<Dockerfile>.dockerignore` (file-specific) |
| Different `.dockerignore` semantics from `.gitignore` for directories | Test with `docker build --no-cache --progress=plain .` |

### Per-Dockerfile Ignore (BuildKit)

```
my-app/
├── Dockerfile
├── Dockerfile.dockerignore          ← scoped to Dockerfile
├── Dockerfile.dev
└── Dockerfile.dev.dockerignore      ← scoped to Dockerfile.dev
```

Lets monorepos define different ignores per service. Falls back to root `.dockerignore` if no per-file one exists.

### Measuring Impact

Before:

```
=> transferring context: 287MB    0.4s
=> CACHED [ 2/12] COPY package.json ./
=> [ 11/12] COPY . .              ← invalidated by .git change
=> [ 12/12] RUN npm run build     ← 90s
```

After a proper `.dockerignore`:

```
=> transferring context: 1.2MB    0.05s
=> CACHED [ 2/12] COPY package.json ./
=> CACHED [11/12] COPY . .
=> CACHED [12/12] RUN npm run build
```

> [!NOTE]
> Treat `.dockerignore` as a security artifact, not just an optimization. A 3-line `.dockerignore` matching `.env*` and `*.key` prevents a whole class of credential leaks.

### Interview Follow-ups

- *"Why doesn't Docker just send only what `COPY` references?"* — Because `COPY` patterns are computed at build time and can use globs. Pre-filtering would change semantics. Also, build args and conditional logic can change what's referenced.
- *"What about `docker build --secret`?"* — BuildKit-only. Mounts a file or env var for one `RUN` step; the file never becomes part of any layer.
- *"Symbolic links in the context?"* — Resolved client-side. A symlink pointing outside the context will fail or be ignored, depending on settings.
