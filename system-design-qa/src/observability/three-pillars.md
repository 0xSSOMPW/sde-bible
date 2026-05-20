# Q: Metrics, logs, traces — the three pillars of observability.

**Answer:**

Three complementary telemetry types:

- **Metrics**: aggregate numbers over time. Cheap, low resolution.
- **Logs**: discrete event records. Rich, expensive.
- **Traces**: causally-linked spans across services. Best for "where did time go?"

Modern observability uses all three, each for what it does best.

### Metrics

Time series of aggregated values:

```
http_requests_total{service="orders", status="200"}  → counter
http_request_duration_seconds                          → histogram
queue_depth                                            → gauge
```

Properties:
- **Cheap**: ~1 byte/sample after compression.
- **Aggregated**: lose per-event detail.
- **Bounded cardinality**: number of unique series.
- **Fast queries** over long time ranges.

Use for: dashboards, SLO tracking, alerting, rate-of-change.

Tools: Prometheus, Datadog, Grafana Cloud, M3DB, VictoriaMetrics.

See [Metrics System](../problems/metrics-system.md).

### Logs

Discrete events with arbitrary structure:

```
{
  "ts": "2025-...",
  "level": "ERROR",
  "service": "orders",
  "trace_id": "...",
  "user_id": "u42",
  "msg": "payment declined",
  "decline_code": "insufficient_funds"
}
```

Properties:
- **High volume**: 1 KB / event typical.
- **Expensive to store**.
- **Rich detail**: full per-event context.
- **Slow to query** over long ranges.

Use for: debugging, audit, forensic analysis, low-frequency events.

Tools: Elasticsearch, Loki, Splunk, Datadog Logs, CloudWatch Logs.

### Traces

A trace = one user request as it flows through services. Each service contributes a **span**; spans are nested:

```
Trace: 5fcd...

  ┌──── span: gateway (200ms)
        │
        ├──── span: orders-service (180ms)
        │      │
        │      ├──── span: db-query (15ms)
        │      └──── span: payments-service (140ms)
        │             │
        │             └──── span: stripe-api (130ms)
        │
        └──── span: emit-event (5ms)
```

Properties:
- Tells you **where time went**.
- Per-request, **high cardinality**.
- Stored sparse (usually sampled).
- Linked to logs via trace ID.

Use for: latency root-cause, distributed debugging.

Tools: Jaeger, Tempo, Datadog APM, Honeycomb, AWS X-Ray.

### When To Use Each

| Question | Pillar |
|----------|--------|
| "What's my error rate?" | Metrics |
| "What's slow today vs last week?" | Metrics |
| "What did this specific failed request do?" | Trace + logs |
| "Why is p99 latency rising?" | Trace (find slow spans) |
| "Did this rare event happen?" | Logs |
| "What's the queue depth right now?" | Metrics |
| "Which user is affected?" | Logs / traces |

Each pillar narrows the search differently. Metrics signal "something is off"; trace shows where; logs reveal what specifically.

### OpenTelemetry

Vendor-neutral SDK + protocol for all three pillars:

```
app instrumented with OTel
   │
   ▼
OTel Collector       (filter, batch, route)
   │
   ├──► metrics → Prometheus
   ├──► traces  → Tempo / Jaeger
   └──► logs    → Loki / ES
```

You write SDK code once; swap backends freely. Modern default for new systems.

### Correlation

The magic is **trace ID propagation**:

```
client → gateway → service A → service B
              │         │         │
              ▼         ▼         ▼
   trace_id: abc trace_id: abc trace_id: abc
```

Every log line, metric, span shares the trace_id. Click a trace → see all logs from that request. See a slow query → find which user.

Standard: W3C Trace Context (`traceparent` header).

### Sampling

100% trace collection is too expensive at scale. Sampling:

- **Head sampling**: gateway decides upfront (e.g., 1% of traffic).
- **Tail sampling**: collect all; decide after seeing the trace (keep errors, slow, ...).

Head sampling is cheap but you might miss the trace that matters. Tail sampling is smarter but needs a buffer (memory) at the collector.

### Cardinality and Cost

| Pillar | Cardinality risk | Cost driver |
|--------|------------------|-------------|
| Metrics | Series count (label combos) | Storage of TSDB |
| Logs | None — every event is a row | Storage + query CPU |
| Traces | Trace IDs (sampled) | Storage of spans |

Metrics break with high-cardinality labels (don't put user_id). Logs and traces tolerate cardinality but cost more.

### SLO-Backed Alerts

Don't alert on every spike. Alert on **SLO burn rate** (see [SLI/SLO/SLA](../fundamentals/slo-sli-sla.md)):

- p99 latency violating SLO for sustained period.
- Error rate exceeds budget burn rate.

Metrics drive these alerts. Traces help root-cause.

### Common Mistakes

| Mistake | Reality |
|---------|---------|
| Replacing metrics with logs ("just grep!") | Logs don't aggregate cheaply; latency comparisons painful |
| Replacing logs with metrics | Can't reconstruct a specific event |
| Logging at INFO every request | Volume explosion; sample or move to traces |
| No trace propagation across services | Can't see the request flow |
| High-cardinality labels in metrics | OOM the TSDB |
| Different log formats per service | Can't query unified |

### Engineering Standard

A production-grade service emits all three:

```
metrics:  
  http_requests_total{service, route, status}
  http_request_duration_seconds_bucket{...}

logs:
  structured JSON with trace_id, request_id, user_id

traces:
  per-RPC spans with attributes (service, method, target)
  errors annotated
```

OpenTelemetry SDK handles all three automatically for typical frameworks.

> [!NOTE]
> The three pillars are not a buffet — they're complementary. Mature teams use all three for distinct purposes. Replacing one with another is a sign of underbuilt observability.

### Interview Follow-ups

- *"What's the cheapest pillar?"* — Metrics, by orders of magnitude. Use them as the alerting layer.
- *"How do you investigate a 'why is the system slow today?' question?"* — Start metrics (find which service / endpoint slowed). Pull traces from that service to find which span. Click into logs of the slow span for detail.
- *"What is eBPF observability?"* — Kernel-level tracing without app instrumentation. Tools: Pixie, Cilium Hubble. Promising; still maturing.
