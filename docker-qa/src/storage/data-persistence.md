# Q: How do you handle Data Persistence in Docker?

**Answer:**

Containers are **ephemeral by default** — all data written inside a container is lost when the container is removed. This is a key interview topic because production systems obviously need persistent data.

### The Problem
```bash
docker run -d --name mydb postgres
# Write data to the database...
docker rm -f mydb
# 💀 All data is gone forever
```

### Strategy 1: Named Volumes (Recommended)
Named volumes persist independently of container lifecycle. Even if the container is removed, the volume survives.

```bash
docker volume create postgres_data
docker run -d --name mydb \
    -v postgres_data:/var/lib/postgresql/data \
    postgres

# Remove the container
docker rm -f mydb

# Data is still safe in the volume!
docker run -d --name mydb-new \
    -v postgres_data:/var/lib/postgresql/data \
    postgres
# New container picks up right where the old one left off
```

### Strategy 2: Docker Compose with Volumes

```yaml
services:
  db:
    image: postgres:16
    volumes:
      - postgres_data:/var/lib/postgresql/data
    environment:
      POSTGRES_PASSWORD: secret

volumes:
  postgres_data:  # Declared as a named volume
```

### Strategy 3: Backup & Restore Volumes

```bash
# Backup a volume to a tar file
docker run --rm \
    -v postgres_data:/data \
    -v $(pwd):/backup \
    alpine tar czf /backup/db-backup.tar.gz -C /data .

# Restore from backup
docker run --rm \
    -v postgres_data:/data \
    -v $(pwd):/backup \
    alpine tar xzf /backup/db-backup.tar.gz -C /data
```

### Strategy 4: External Storage Drivers
For production clusters, volume drivers allow Docker to store data on external systems:
- **AWS EBS** volumes
- **NFS** shares
- **GlusterFS** / **Ceph** distributed filesystems

```bash
docker volume create --driver rexray/ebs myvolume
```

> [!TIP]
> In interviews, mentioning backup strategies and external volume drivers shows production-level thinking beyond basic `docker run -v`.
