# Q: Tags vs digests — why should production images be pinned by digest?

**Answer:**

Tags are *mutable labels*; digests are *immutable content hashes*. Deploying `nginx:1.25` today and `nginx:1.25` next week can give you two different images with the same name. Pinning by digest is the difference between deterministic builds and a supply-chain incident waiting to happen.

### What Each One Is

```
nginx:1.25                                       <-- tag
nginx@sha256:9b6a3f51c4e6e9a7...c9                <-- digest
nginx:1.25@sha256:9b6a3f51c4e6e9a7...c9           <-- both (recommended)
```

- **Tag**: a pointer in the registry to a manifest. Mutable. The registry operator (or anyone with push access) can move it.
- **Digest**: the SHA-256 of the image manifest. Content-addressable. Two identical manifests anywhere in the world produce the same digest.

### Why Tags Are Dangerous

```
Day 1: nginx:1.25 → digest A
Day 2: nginx:1.25 → digest A (still — patch release pending)
Day 8: nginx:1.25 → digest B (1.25.4 silently replaces 1.25.3)

Your CI:
  Day 1 deploy works.
  Day 8 deploy ships a new binary into prod with no code change in your repo.
```

The same is true even for "version" tags:
- `1.25` (floats over patch releases)
- `1` (floats over minor releases)
- `latest` (floats over everything; pretend it doesn't exist)
- Internal teams routinely overwrite `:prod`, `:stable`, `:v1` tags.

### Pinning by Digest

```dockerfile
FROM nginx:1.25.3@sha256:9b6a3f51c4e6e9a7b5e9b8c1d2a3b4c5d6e7f8a9b0c1d2e3f4a5b6c7d8e9f0
```

- The tag is documentation for humans.
- The digest is what the daemon resolves and verifies.
- Even if someone retags `1.25.3` to point at malware, your build is unaffected — the digest mismatch fails the pull.

### Where Digests Live

```bash
docker pull nginx:1.25
# 1.25: Pulling from library/nginx
# Digest: sha256:9b6a3f51c4e6e9a7b5e9b8c1d2a3b4c5d6e7f8a9b0c1d2e3f4a5b6c7d8e9f0

docker inspect --format='{{index .RepoDigests 0}}' nginx:1.25
# nginx@sha256:9b6a3f51c4...
```

Get it from a registry without pulling:

```bash
docker buildx imagetools inspect nginx:1.25
# Or:
crane digest nginx:1.25
```

### What This Means for Workflows

| Situation | Use |
|-----------|-----|
| Top of `Dockerfile` (FROM) | Digest pin |
| Compose / Helm / K8s manifests | Digest pin (`image: app@sha256:...`) |
| Local dev iteration | Tag is fine |
| Internal "rolling" base images | Tag with `imagePullPolicy: Always` is acceptable if controlled |
| Reproducible builds (SBOM, supply chain) | Digest pin everywhere |

### Multi-Arch Quirk

For a multi-platform image, the digest of `nginx:1.25` refers to a **manifest list** (the index), not a single platform's image. Pulling on amd64 resolves to a different *layer* digest than on arm64, but the top-level manifest digest is the same across platforms — which is what you want when pinning.

```bash
docker buildx imagetools inspect nginx:1.25 --raw
# Shows the manifest list with per-platform digests inside.
```

### Automation: Digest Pinning Tools

- **renovate**, **dependabot** — open PRs to bump pinned digests while keeping a human-readable tag.
- `docker buildx imagetools inspect` to fetch the current digest.
- For Helm/K8s: `kbld`, `kustomize edit set image`, or your CD's templating.

Example renovate config:

```json
{
  "docker": {
    "pinDigests": true
  }
}
```

### Push-Side: Why Digests Help Deployments Too

A K8s `Deployment` referencing `app:prod` won't redeploy when you push a new image to that tag — same `image`, same spec hash, nothing to roll. Tooling either:

1. Pushes by digest (`app@sha256:...`) — changes the spec, triggers rollout.
2. Or uses `imagePullPolicy: Always` and bumps a label/annotation.

Digest-by-default makes (1) automatic.

### Common Pitfalls

| Pitfall | Fix |
|---------|-----|
| Mixing `:latest` with retention policies in CI | Always tag with immutable identifier (git SHA) AND push to a moving tag if needed |
| Digest pin in `FROM`, but base image not in registry yet | CI must pull from the same registry the digest was computed against |
| Pinning to a digest that gets garbage-collected | Use an immutable retention policy in the registry, or mirror to your own |
| Believing `latest` is "the latest" | It's whatever the last push named `latest`, nothing more |

> [!NOTE]
> Digests are SHA-256 over the manifest, not the image bytes. Changing a label or build-arg metadata changes the digest even if the actual filesystem layers are identical. That's a feature: it lets you audit *what* was built, not just *what bytes* it contained.

### Interview Follow-ups

- *"Can two different images have the same digest?"* — No (collision resistance of SHA-256). The point of content addressing.
- *"What's the difference between manifest digest and image ID?"* — Manifest digest is registry-side, over the manifest JSON. Image ID is local-daemon-side, over the config JSON. They're related but distinct.
- *"What is Notary / Cosign?"* — Tools to sign image digests; verification happens against the digest, not the tag, for exactly this reason.
