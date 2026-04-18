# Q: What are Docker Logging Best Practices?

**Answer:**

Proper logging is essential for debugging, monitoring, and auditing containerized applications.

### The Golden Rule: Log to stdout/stderr
Docker captures everything written to the container's **stdout** and **stderr** streams. Applications should NOT write logs to files inside the container.

```dockerfile
# ❌ Bad: Logs trapped inside the container filesystem
CMD ["node", "index.js", ">>", "/var/log/app.log"]

# ✅ Good: Logs go to stdout (Docker captures them)
CMD ["node", "index.js"]
```

### Why stdout/stderr?
1. **`docker logs`** only shows stdout/stderr output.
2. Log drivers can only capture stdout/stderr.
3. Files inside the container are lost when the container is removed.
4. Centralized logging systems (ELK, Datadog, CloudWatch) integrate with Docker's log drivers, not container files.

### Docker Log Drivers

Docker supports pluggable logging drivers that determine where container logs are sent:

```bash
# View current log driver
docker info --format '{{.LoggingDriver}}'

# Run a container with a specific driver
docker run --log-driver=json-file --log-opt max-size=10m --log-opt max-file=3 myapp
```

| Driver | Destination |
|---|---|
| `json-file` | Local JSON files (default) |
| `syslog` | Syslog daemon |
| `fluentd` | Fluentd collector |
| `awslogs` | AWS CloudWatch |
| `gcplogs` | Google Cloud Logging |
| `splunk` | Splunk HTTP Event Collector |
| `none` | Discard all logs |

### Log Rotation (Critical!)
The default `json-file` driver has **no size limit**. Logs will grow until they fill the disk.

```json
// /etc/docker/daemon.json
{
    "log-driver": "json-file",
    "log-opts": {
        "max-size": "10m",
        "max-file": "3"
    }
}
```

In Compose:
```yaml
services:
  api:
    image: myapp
    logging:
      driver: json-file
      options:
        max-size: "10m"
        max-file: "3"
```

> [!CAUTION]
> Forgetting log rotation is one of the most common causes of production outages in Docker environments. A single chatty container can fill up the host's disk in hours.

### Useful Commands

```bash
# View logs
docker logs mycontainer

# Follow logs (like tail -f)
docker logs -f mycontainer

# Show last 100 lines
docker logs --tail 100 mycontainer

# Show logs since a timestamp
docker logs --since 2024-01-01T00:00:00 mycontainer
```
