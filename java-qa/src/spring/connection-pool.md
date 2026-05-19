# Q: How do you tune HikariCP, and why is connection pool sizing critical?

**Answer:**

HikariCP is Spring Boot's default JDBC pool. Misconfiguring it is the most common cause of production database incidents: connection leaks, exhausted pools, slow startup, and "too many connections" errors at the DB. The rules are counterintuitive — more connections usually means *worse* performance.

### Why You Need a Pool

```
Without pool:
  Each request → open TCP + TLS + auth handshake → run query → close
  Cost: 50–200 ms per connection on Postgres/MySQL
  Throughput: limited by handshake, not query

With pool:
  Long-lived connections kept warm
  Request borrows one, returns it
  Cost amortized to near zero
```

### Key Settings

```yaml
spring:
  datasource:
    hikari:
      maximum-pool-size: 10
      minimum-idle: 10
      connection-timeout: 3000        # ms to wait for a connection
      idle-timeout: 600000            # ms before idle conn is closed (10m)
      max-lifetime: 1800000           # ms max conn age (30m)
      leak-detection-threshold: 30000 # warn if borrow > 30s
      validation-timeout: 5000
      connection-init-sql: "SELECT 1"
```

### The Sizing Counterintuition

> "If 10 connections handle 1000 req/s, 100 connections should handle 10000 req/s."

**Wrong.** Beyond a small number, more connections **slow you down** because:
- Each DB connection has a thread/process on the DB side.
- More DB threads → more context switching, more lock contention, more cache misses.
- Postgres in particular: each backend is a process, ~5–10 MB RAM each.

Empirical guidance from HikariCP authors:

```
connections = ((core_count * 2) + effective_spindle_count)
```

For a modern SSD-backed DB with 8 cores: `8*2 + 1 ≈ 17`. Use **10–20**, not 100.

### Why Bigger Pools Hurt

Consider a DB with 200 max connections. App pool = 200:
- Connection-borrow latency: zero (always one free).
- Query latency: high — every query fights 199 others for CPU, locks, buffer cache.

App pool = 20:
- Connection-borrow latency: low, occasional brief wait.
- Query latency: low — only 20 queries in flight, DB happy.
- Throughput: higher despite the queueing.

This is **Little's Law** at work. Latency × throughput = concurrent work. Lower the concurrent work, and latency drops faster than throughput rises.

### Per-Instance Sizing in Microservices

Each app instance has its own pool. If you have:
- DB max_connections = 200.
- 10 app instances.
- 20 admin/reporting connections reserved.

Per-instance pool = `(200 - 20) / 10 = 18` → round to 15 for safety.

A common production failure: autoscaling from 10 → 50 pods, each with `maximum-pool-size: 20` → 1000 connection requests at a 200-conn DB. Pods fail health checks waiting on `connection-timeout`.

### Critical Settings Explained

**`maximum-pool-size`**: hard cap. Default 10. Don't blindly increase.

**`minimum-idle`**: HikariCP authors recommend setting this **equal to max** for a fixed-size pool, avoiding the cost of ramping up under load.

**`connection-timeout`**: how long a thread waits for a connection before throwing `SQLException`. Default 30s — usually too long. Use 1–5s in user-facing services so callers fail fast and load balancers can shed.

**`max-lifetime`**: rotates connections every N ms. Critical for:
- Picking up DB failover events (old conn pointing at dead replica).
- Working around stateful proxies (RDS Proxy, PgBouncer transaction pool).
- Set to less than your DB's idle timeout (typically 30 min vs DB's 60 min).

**`idle-timeout`**: closes idle connections after this duration. Set 0 to disable for fixed-size pools.

**`leak-detection-threshold`**: prints a stack trace if a borrowed connection isn't returned in N ms. **Enable this in production** — 30s catches most leaks without false positives.

### Connection Leaks

```java
// ❌ Leak: connection never closed on exception
public void process() throws SQLException {
    Connection c = ds.getConnection();
    PreparedStatement s = c.prepareStatement("SELECT ...");
    if (someCondition) throw new RuntimeException();   // c leaked!
    s.executeQuery();
    c.close();
}

// ✅ try-with-resources guarantees cleanup
public void process() throws SQLException {
    try (Connection c = ds.getConnection();
         PreparedStatement s = c.prepareStatement("SELECT ...")) {
        s.executeQuery();
    }
}
```

Spring's `JdbcTemplate`, JPA, and `TransactionTemplate` handle this for you. Raw JDBC requires discipline.

### `@Transactional` and Pool Behavior

```java
@Transactional
public void slow() {
    Order o = repo.findById(1L).orElseThrow();
    callExternalAPI(o);                  // ← holds DB conn during HTTP call
    repo.save(o);
}
```

The connection is borrowed at transaction start and held until commit/rollback. A 30-second external call → 30-second connection occupancy → pool saturates under load.

Fix:

```java
public void slow() {
    Order o = readTx(() -> repo.findById(1L).orElseThrow());
    callExternalAPI(o);                  // no DB conn during HTTP call
    writeTx(() -> repo.save(o));
}
```

Never hold a transaction across a remote call.

### Read-Only Pool Pattern

Separate read traffic to a replica with a second `DataSource`:

```java
@Bean @Primary
DataSource primary() { return hikari(primaryProps()); }

@Bean
DataSource replica() { return hikari(replicaProps()); }
```

Cuts load on the primary and keeps writes isolated from analytical reads.

### Observing the Pool

HikariCP exposes Micrometer metrics out of the box:

```
hikaricp.connections.active    # in-use right now
hikaricp.connections.idle      # waiting in pool
hikaricp.connections.pending   # threads waiting to borrow (>0 is a smell)
hikaricp.connections.timeout   # cumulative borrow timeouts
hikaricp.connections.usage     # histogram of how long borrowed
```

Alert on:
- `pending > 0` for sustained periods.
- `active ≈ max` consistently.
- `timeout` rate > 0.

### Common Mistakes

| Mistake | Fix |
|---------|-----|
| `maximum-pool-size: 100` "for safety" | 10–20 is right for most workloads |
| Holding `@Transactional` across HTTP/RPC | Split the transaction, use compensations |
| Different pools sharing one DB max_connections, oversubscribed | Plan pool budget org-wide; reserve admin slots |
| `connection-timeout: 30000` user-facing | Cut to 1–5 s; fail fast |
| Forgot to close raw JDBC `Connection`/`Statement`/`ResultSet` | Use try-with-resources |
| `max-lifetime` > DB idle timeout | Pool hands out dead connections; set < DB limit |

> [!NOTE]
> If a problem looks like "we need more connections," it usually means a query is slow, a transaction is too long, or pool is misconfigured. More connections is rarely the cure.

### Interview Follow-ups

- *"Why does HikariCP outperform DBCP/c3p0?"* — Smaller code, no synchronization on the hot path, FastList instead of ArrayList for the connection bag, careful CAS-based state transitions.
- *"What's PgBouncer and how does it interact?"* — A connection multiplexer in front of Postgres. Two modes: session (1:1 like a real pool) and transaction (connection released to next client per transaction — incompatible with prepared statements and `SET LOCAL`).
- *"How would you tune for serverless functions?"* — Each instance is short-lived; either use a single connection per instance and rely on the DB-side proxy (RDS Proxy, Neon, PlanetScale) or pre-warm a small pool of 1–2 connections.
