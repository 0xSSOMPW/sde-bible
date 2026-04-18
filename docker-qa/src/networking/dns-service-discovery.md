# Q: How does Container DNS and Service Discovery work in Docker?

**Answer:**

Docker has a built-in DNS server that enables containers to discover and communicate with each other by name instead of IP address.

### How It Works
When you create a **custom bridge network**, Docker runs an embedded DNS server at `127.0.0.11`. Every container on that network automatically registers its container name as a DNS hostname.

```bash
docker network create backend
docker run -d --name api --network backend myapi
docker run -d --name db --network backend postgres

# Inside the "api" container:
ping db           # Resolves to the postgres container's IP ✅
curl http://db:5432  # Works by name ✅
```

### Default Bridge vs Custom Bridge

| Feature | Default Bridge | Custom Bridge |
|---|---|---|
| DNS resolution by name | ❌ No | ✅ Yes |
| Container isolation | Shared with all default containers | Isolated per network |
| Legacy `--link` needed? | Yes (deprecated) | No |

> [!WARNING]
> The `--link` flag is **deprecated**. Always use custom bridge networks for container-to-container communication.

### Network Aliases
You can give a container multiple DNS names using `--network-alias`:

```bash
docker run -d --name postgres-primary \
    --network backend \
    --network-alias db \
    --network-alias database \
    postgres

# Other containers can reach it via "postgres-primary", "db", OR "database"
```

### Docker Compose — Automatic Service Discovery
In Docker Compose, each service name automatically becomes a DNS hostname on the shared network.

```yaml
services:
  api:
    build: ./api
    depends_on:
      - db
  db:
    image: postgres:16
```

Inside the `api` container, `db` resolves to the Postgres container. No manual network configuration needed.

### Round-Robin DNS (Load Balancing)
If multiple containers share the same network alias, Docker's DNS returns all their IPs in a round-robin fashion:

```bash
docker run -d --network backend --network-alias worker myworker
docker run -d --network backend --network-alias worker myworker
docker run -d --network backend --network-alias worker myworker

# Resolving "worker" returns all 3 IPs, rotating order each time
```
