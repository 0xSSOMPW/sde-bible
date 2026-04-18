# Q: What is the difference between `depends_on` and health checks in Docker Compose?

**Answer:**

This is a subtle but extremely important question. `depends_on` controls **startup order** but does NOT wait for a service to be **ready**.

### `depends_on` (Startup Order Only)
By default, `depends_on` only guarantees that the dependent container has **started** (i.e., `docker run` has been called). It does NOT wait for the application inside to be fully initialized and accepting connections.

```yaml
services:
  api:
    build: ./api
    depends_on:
      - db   # db container STARTS first, but may not be ready yet!
  db:
    image: postgres:16
```

**The Problem:** Postgres takes several seconds to initialize. Your API container starts immediately after the Postgres container starts, but the database isn't accepting connections yet. The API crashes with "connection refused."

### Health Checks (Readiness Verification)
A health check defines a command that Docker runs periodically to determine if a container is actually **healthy** (i.e., the application inside is ready).

```yaml
services:
  db:
    image: postgres:16
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      timeout: 5s
      retries: 5
      start_period: 10s
```

### The Solution: `depends_on` + `condition`
Combine both to make a service wait until its dependency is truly healthy:

```yaml
services:
  api:
    build: ./api
    depends_on:
      db:
        condition: service_healthy  # Wait until db passes health check
  db:
    image: postgres:16
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      timeout: 5s
      retries: 5
      start_period: 10s
```

Now the `api` service will **not start** until the `db` service passes its health check.

### Health Check Conditions

| Condition | Meaning |
|---|---|
| `service_started` | Default. Container has started (same as basic `depends_on`). |
| `service_healthy` | Container's health check is passing. |
| `service_completed_successfully` | Container ran and exited with code 0 (for init/migration containers). |

> [!TIP]
> The `service_completed_successfully` condition is perfect for running database migrations before starting the API:
> ```yaml
> migrate:
>   image: myapp
>   command: npm run migrate
> api:
>   depends_on:
>     migrate:
>       condition: service_completed_successfully
> ```
