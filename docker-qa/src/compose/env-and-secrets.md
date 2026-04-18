# Q: How do you manage Environment Variables and Secrets in Docker?

**Answer:**

Environment variables are the primary way to configure containerized applications. However, sensitive data (passwords, API keys) requires special handling.

### 1. Inline Environment Variables
Pass variables directly in `docker run`:
```bash
docker run -e DATABASE_URL=postgres://localhost:5432/mydb myapp
```

In Compose:
```yaml
services:
  api:
    environment:
      - DATABASE_URL=postgres://db:5432/mydb
      - NODE_ENV=production
```

### 2. `.env` Files
Store variables in a file and load them:
```bash
docker run --env-file .env myapp
```

In Compose, `.env` in the project root is automatically loaded for variable substitution:
```yaml
# .env
POSTGRES_PASSWORD=supersecret
DB_PORT=5432

# compose.yaml
services:
  db:
    image: postgres:16
    ports:
      - "${DB_PORT}:5432"
    environment:
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
```

### 3. `env_file` directive
Load variables from a specific file into the container's environment:
```yaml
services:
  api:
    env_file:
      - ./config/.env.production
```

> [!CAUTION]
> **Never commit `.env` files** with real secrets to version control. Add them to `.gitignore` and `.dockerignore`.

### 4. Docker Secrets (Swarm Mode)
For true secret management, Docker Swarm provides encrypted secrets that are mounted as files inside containers (never exposed as environment variables or stored in image layers).

```bash
echo "supersecretpassword" | docker secret create db_password -

# Use in a Swarm service
docker service create \
    --secret db_password \
    --name myapp myimage
# Secret is available at /run/secrets/db_password inside the container
```

In Compose (with Swarm):
```yaml
services:
  api:
    secrets:
      - db_password

secrets:
  db_password:
    file: ./secrets/db_password.txt
```

### Security Hierarchy (Least to Most Secure)
1. ❌ Hardcoded in Dockerfile (`ENV PASSWORD=secret`) — visible in image layers.
2. ⚠️ `-e` flag or `environment:` in Compose — visible in `docker inspect`.
3. ✅ `--env-file` — secrets in a gitignored file, but still visible in `docker inspect`.
4. ✅✅ Docker Secrets — encrypted at rest, mounted as tmpfs, not in `docker inspect`.
5. ✅✅✅ External vault (HashiCorp Vault, AWS Secrets Manager) — most secure for production.
