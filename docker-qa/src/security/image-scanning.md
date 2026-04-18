# Q: How do you scan Docker images for vulnerabilities and minimize the attack surface?

**Answer:**

Container image security is a critical production concern. Vulnerabilities in base images, dependencies, or OS packages can be exploited.

### 1. Image Scanning Tools
Scan images for known CVEs (Common Vulnerabilities and Exposures):

```bash
# Docker Scout (built into Docker Desktop)
docker scout cves myimage:latest

# Trivy (open-source, most popular)
trivy image myimage:latest

# Snyk
snyk container test myimage:latest

# Grype (by Anchore)
grype myimage:latest
```

### 2. Minimizing the Attack Surface

**Use minimal base images:**
```dockerfile
# ❌ Full OS (~900MB, thousands of packages)
FROM node:20

# ✅ Alpine (~130MB, minimal packages)
FROM node:20-alpine

# ✅✅ Distroless (~20MB, no shell, no package manager)
FROM gcr.io/distroless/nodejs20-debian12
```

**Why Distroless?**
Distroless images contain *only* the application runtime. There's no shell (`/bin/sh`), no package manager, no utilities. If an attacker gets inside the container, they can't run `curl`, `wget`, or even `ls`.

**Multi-stage builds** (see the Images chapter) are essential for keeping build tools out of production images.

### 3. Never Use `latest` Tag
```dockerfile
# ❌ Bad: "latest" could change at any time
FROM node:latest

# ✅ Good: Pinned digest for reproducibility
FROM node:20.11-alpine3.18@sha256:abc123...
```

### 4. Scan in CI/CD Pipeline

```yaml
# GitHub Actions example
- name: Scan image
  uses: aquasecurity/trivy-action@master
  with:
    image-ref: myimage:${{ github.sha }}
    severity: CRITICAL,HIGH
    exit-code: 1  # Fail the build if critical vulnerabilities found
```

### 5. Don't Store Secrets in Images

```dockerfile
# ❌ TERRIBLE: Secret is baked into a layer permanently
COPY .env /app/.env
ENV API_KEY=sk-abc123

# ✅ Pass secrets at runtime
docker run -e API_KEY=$API_KEY myimage
```

> [!CAUTION]
> Even if you delete a secret in a later Dockerfile layer, it still exists in the earlier layer and can be extracted with `docker history` or by inspecting the image layers directly.

### Checklist
- [ ] Use minimal base images (Alpine, Distroless, or Scratch)
- [ ] Run as non-root user
- [ ] Scan images in CI with Trivy or Scout
- [ ] Pin base image versions (avoid `latest`)
- [ ] No secrets in image layers
- [ ] Use multi-stage builds
- [ ] Drop unnecessary Linux capabilities
- [ ] Use read-only filesystem where possible
