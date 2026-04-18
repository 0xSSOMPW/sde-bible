# Q: What is Docker Compose and when would you use it?

**Answer:**

**Docker Compose** is a tool for defining and running **multi-container applications** using a single YAML configuration file (`docker-compose.yml` or `compose.yaml`). Instead of running multiple `docker run` commands with complex flags, you declare everything in one file and spin up the entire stack with a single command.

### Without Compose (Painful)
```bash
docker network create myapp
docker volume create db_data
docker run -d --name db --network myapp -v db_data:/var/lib/postgresql/data \
    -e POSTGRES_PASSWORD=secret postgres:16
docker run -d --name redis --network myapp redis:7
docker run -d --name api --network myapp -p 3000:3000 \
    -e DATABASE_URL=postgres://db:5432 \
    -e REDIS_URL=redis://redis:6379 myapi
```

### With Compose (Clean)
```yaml
# compose.yaml
services:
  api:
    build: ./api
    ports:
      - "3000:3000"
    environment:
      DATABASE_URL: postgres://db:5432/mydb
      REDIS_URL: redis://redis:6379
    depends_on:
      - db
      - redis

  db:
    image: postgres:16
    volumes:
      - db_data:/var/lib/postgresql/data
    environment:
      POSTGRES_PASSWORD: secret

  redis:
    image: redis:7-alpine

volumes:
  db_data:
```

```bash
# Start everything
docker compose up -d

# View logs
docker compose logs -f api

# Stop and remove everything
docker compose down

# Stop, remove, AND delete volumes (nuclear option)
docker compose down -v
```

### Key Features
- **Automatic networking**: All services in a `compose.yaml` automatically join a shared network and can reach each other by service name.
- **Volume management**: Volumes are declared and managed alongside services.
- **Build integration**: You can specify `build:` context directly instead of pre-building images.
- **Profiles**: Group services into profiles for conditional startup.
- **Override files**: Use `compose.override.yaml` for environment-specific config.

### When to Use Compose
- **Local development**: Spin up your full stack (API + DB + cache + queue) in one command.
- **CI/CD**: Run integration tests against real services.
- **Single-host production**: Small deployments that don't need Kubernetes.

> [!NOTE]
> Docker Compose is **not** an orchestration tool. It runs containers on a single host. For multi-host orchestration, use Docker Swarm or Kubernetes.
