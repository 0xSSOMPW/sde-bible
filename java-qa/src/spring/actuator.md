# Q: What is Spring Boot Actuator? What endpoints matter in production?

**Answer:**

Actuator = production-ready monitoring/management endpoints over HTTP (or JMX): health, metrics, info, env, mappings, etc.

### Add It
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

Default exposed: `/actuator/health` and `/actuator/info` over HTTP. Everything else: JMX-only by default, must opt in.

### Expose Endpoints
```yaml
management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,prometheus,loggers
        # exclude: env  # hide sensitive ones if include=*
  endpoint:
    health:
      show-details: when_authorized   # never | always | when_authorized
```

### Key Endpoints

| Endpoint | Use |
|---|---|
| `/actuator/health` | Liveness/readiness probes (k8s) |
| `/actuator/info` | Build info, git commit |
| `/actuator/metrics` | Counters, gauges, timers (Micrometer) |
| `/actuator/prometheus` | Prometheus-format metrics |
| `/actuator/env` | All resolved properties (sensitive!) |
| `/actuator/configprops` | `@ConfigurationProperties` beans |
| `/actuator/beans` | Bean graph |
| `/actuator/mappings` | URL → handler mappings |
| `/actuator/loggers` | View / **change** log levels at runtime |
| `/actuator/threaddump` | Live thread dump |
| `/actuator/heapdump` | Download `.hprof` |
| `/actuator/httpexchanges` | Recent HTTP requests |
| `/actuator/scheduledtasks` | `@Scheduled` registry |
| `/actuator/shutdown` | Graceful shutdown (disabled by default) |

### Health
```json
{
  "status": "UP",
  "components": {
    "db": {"status":"UP","details":{"database":"PostgreSQL","validationQuery":"isValid()"}},
    "diskSpace": {"status":"UP"},
    "redis": {"status":"UP"}
  }
}
```

Custom indicator:
```java
@Component
public class StripeHealthIndicator implements HealthIndicator {
    @Override
    public Health health() {
        try {
            stripe.ping();
            return Health.up().build();
        } catch (Exception e) {
            return Health.down(e).build();
        }
    }
}
```

### Liveness vs Readiness (k8s)
Boot exposes both groups under `/actuator/health`:
- `/actuator/health/liveness` — is the app alive?
- `/actuator/health/readiness` — should it receive traffic?

```yaml
management:
  endpoint:
    health:
      probes:
        enabled: true
```

### Metrics (Micrometer)
Boot wires Micrometer in. Auto-registers JVM, system, HTTP, JDBC, JPA, Tomcat metrics.

Custom:
```java
@RestController
class OrderController {
    private final Counter ordersPlaced;

    OrderController(MeterRegistry r) {
        this.ordersPlaced = Counter.builder("orders.placed")
            .tag("region", "us-east").register(r);
    }

    @PostMapping("/orders")
    void place(@RequestBody Order o) {
        ordersPlaced.increment();
        ...
    }
}

// Timer
@Timed(value = "orders.process.time", percentiles = {0.5, 0.95, 0.99})
public void process(Order o) { ... }
```

Backends: Prometheus, Datadog, CloudWatch, New Relic, Graphite — add the right `micrometer-registry-*` dependency.

### Securing Actuator
Sensitive endpoints (env, heapdump, loggers, beans) leak config + memory. Lock them:

```java
@Bean
SecurityFilterChain actuatorSecurity(HttpSecurity http) throws Exception {
    return http
        .securityMatcher(EndpointRequest.toAnyEndpoint())
        .authorizeHttpRequests(a -> a
            .requestMatchers(EndpointRequest.to("health", "info")).permitAll()
            .anyRequest().hasRole("ADMIN"))
        .httpBasic(withDefaults())
        .build();
}
```

Or run actuator on a **separate port** internal-only:
```yaml
management:
  server:
    port: 9090
    address: 127.0.0.1
```

### Production Recipe
- Expose: `health`, `info`, `metrics`, `prometheus`.
- Lock everything else behind auth or internal port.
- Wire `/actuator/prometheus` into Prometheus scrape.
- k8s probes → liveness/readiness groups.
- Build info via `spring-boot-maven-plugin` `build-info` goal → shows in `/actuator/info`.

### Useful info Contributors
- Git commit (`spring-boot-starter-actuator` + `git-commit-id-plugin`).
- Build time/version (Maven plugin `build-info`).
- Custom: implement `InfoContributor`.
