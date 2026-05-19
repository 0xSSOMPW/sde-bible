# Q: How do you instrument a Spring Boot app with Micrometer (metrics, traces, logs)?

**Answer:**

Observability has three pillars: **metrics**, **traces**, **logs**. Spring Boot ships Micrometer for metrics + tracing and supports any backend (Prometheus, Datadog, New Relic, Tempo, etc.) via a façade pattern similar to SLF4J's role for logging.

### The Stack

```
Application code
       │
       ▼
   Micrometer API (vendor-neutral)
       │
       ▼
   Registry (Prometheus / OTLP / Datadog / ...)
       │
       ▼
   Backend (Grafana, Jaeger, Tempo, DD, ...)
```

You write `meterRegistry.counter(...)` once. The backend is swappable via dependency.

### Setup

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-registry-prometheus</artifactId>
</dependency>
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-tracing-bridge-otel</artifactId>
</dependency>
<dependency>
    <groupId>io.opentelemetry</groupId>
    <artifactId>opentelemetry-exporter-otlp</artifactId>
</dependency>
```

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health,info,prometheus,metrics
  metrics:
    tags:
      application: ${spring.application.name}
      env: ${ENV:dev}
  tracing:
    sampling:
      probability: 0.1            # sample 10%
  otlp:
    tracing:
      endpoint: http://otel-collector:4318/v1/traces
```

### Built-In Metrics

Spring Boot auto-instruments:

- **HTTP server**: `http.server.requests` (timer per URI/method/status).
- **HTTP client** (`RestClient`, `WebClient`): `http.client.requests`.
- **JDBC pool**: `hikaricp.connections.*`.
- **JVM**: `jvm.memory.used`, `jvm.gc.pause`, `jvm.threads.live`.
- **Kafka**: producer/consumer metrics via `KafkaClientMetrics`.
- **Tomcat/Jetty**: request rate, thread pool stats.

Scrape from Prometheus:

```
GET /actuator/prometheus
```

### Custom Metrics

Use the right type:

| Type | Use | Example |
|------|-----|---------|
| `Counter` | Monotonic event counts | `orders.placed` |
| `Gauge` | Snapshot value | `queue.size` |
| `Timer` | Duration histograms | `payment.processing` |
| `DistributionSummary` | Non-time distributions | `request.size.bytes` |

```java
@Service
class PaymentService {
    private final Counter placed;
    private final Timer processing;

    PaymentService(MeterRegistry r) {
        placed = Counter.builder("payments.placed")
                        .description("Payments accepted")
                        .tag("region", "us-east-1")
                        .register(r);
        processing = Timer.builder("payment.processing")
                          .publishPercentileHistogram()
                          .register(r);
    }

    public void pay(Payment p) {
        processing.record(() -> {
            placed.increment();
            // ...
        });
    }
}
```

### Cardinality — The Production Killer

**Never** tag with high-cardinality values:

```java
counter.tag("user_id", userId)      // ❌ explodes — one series per user
counter.tag("trace_id", traceId)    // ❌ same
counter.tag("path", request.uri())  // ❌ /orders/123, /orders/124, ...
```

```java
counter.tag("region", "us-east-1")  // ✅ bounded
counter.tag("status", "200")         // ✅
counter.tag("path", "/orders/{id}")  // ✅ templated
```

Spring's HTTP timer **uses the templated URI** automatically. If you build URIs manually with IDs, you'll explode cardinality.

### Histograms / Percentiles

```java
Timer.builder("http.api")
     .publishPercentileHistogram()                // export histogram buckets
     .serviceLevelObjectives(
         Duration.ofMillis(100),
         Duration.ofMillis(500),
         Duration.ofSeconds(1))
     .register(registry);
```

`publishPercentileHistogram` is the right answer in 95% of cases — Prometheus computes percentiles across instances. Don't use `publishPercentiles` (client-side estimate; can't aggregate).

### Distributed Tracing

Spring Boot 3 + Micrometer Tracing replaces Sleuth. Auto-propagates `traceparent` headers via:

- `RestClient` / `WebClient`
- `RestTemplate`
- `KafkaTemplate` / `@KafkaListener`
- Reactor schedulers
- `@Async`

Manual span:

```java
@Service
class Quoter {
    private final ObservationRegistry observations;

    String quote(String sku) {
        return Observation.createNotStarted("quote.lookup", observations)
            .lowCardinalityKeyValue("sku.kind", classify(sku))
            .observe(() -> doExpensiveLookup(sku));
    }
}
```

`Observation` produces both a metric (a timer) and a span automatically — the unified API.

### Logging Correlation

With Micrometer Tracing on the classpath, MDC automatically gets `traceId` and `spanId`. Pattern:

```yaml
logging:
  pattern:
    level: "%5p [${spring.application.name},%X{traceId:-},%X{spanId:-}]"
```

Log line:

```
2026-05-18 12:00:00 INFO [orders,a1b2c3d4...e9,1234] OrderService - order placed
```

Now any log line links to its trace.

### Health Checks

```yaml
management:
  endpoint:
    health:
      show-details: when-authorized
      probes:
        enabled: true            # K8s liveness + readiness endpoints
```

Endpoints:

```
GET /actuator/health/liveness    # is the app alive?
GET /actuator/health/readiness   # can it serve traffic?
```

Custom indicator:

```java
@Component
class KafkaHealth implements HealthIndicator {
    public Health health() {
        return reachable() ? Health.up().build() : Health.down().withDetail("err", "...").build();
    }
}
```

Readiness should fail if downstreams (DB, queue) are not ready. Liveness should only fail if the app is *truly broken* (stuck thread, deadlock) — never on transient downstream issues, or K8s will restart-loop the pod.

### What to Alert On (SLOs)

- **Availability**: `sum(rate(http_server_requests_seconds_count{status=~"5.."}[5m])) / sum(rate(http_server_requests_seconds_count[5m]))`
- **Latency**: p99 of `http_server_requests_seconds`
- **Saturation**: `hikaricp_connections_pending`, `jvm_threads_live > N`, `kafka_consumer_records_lag_max`

Pick a few SLIs; alert on burn rate, not raw thresholds (Google SRE workbook).

### Common Mistakes

| Mistake | Fix |
|---------|-----|
| `Tag.of("user", userId)` | Cardinality explosion |
| 100% trace sampling in prod | Sampling overhead crushes throughput |
| Liveness probe = readiness probe | Liveness should rarely fail |
| Logging full payloads at INFO | Cost + PII risk; sample or DEBUG |
| Custom timer with no histogram | Can't aggregate percentiles cross-instance |
| Forgetting to register meters as singletons | Per-request meters silently leak |

> [!NOTE]
> If you can't answer "what's our error rate right now?" in 30 seconds, you don't have observability. Defaults + a handful of custom timers cover 80% of operational questions.

### Interview Follow-ups

- *"Difference between Micrometer Tracing and OpenTelemetry?"* — Micrometer Tracing is the API; OTel is one of its implementations. You can also bridge to Brave (Zipkin).
- *"Why histograms over percentiles?"* — Percentiles can't be averaged or combined across instances. Histograms can (`histogram_quantile`).
- *"How do you trace through Kafka?"* — `traceparent` propagated in Kafka record headers via Spring's `KafkaTemplate` interceptor + listener container.
