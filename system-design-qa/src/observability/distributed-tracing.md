# Q: Distributed tracing — how it works, what to watch for.

**Answer:**

A trace follows one request across services. Implemented via **propagating a trace context** through every call. Critical for understanding latency and failures in microservice systems.

### The Data Model

```
Trace = collection of Spans

Span:
  trace_id        (unique per request)
  span_id         (unique per operation)
  parent_span_id  (causal parent)
  name            ("GET /orders/123")
  start_time
  end_time
  attributes      ({ http.status: 200, db.statement: ... })
  events          (logs scoped to this span)
  status          (OK / Error)
```

Spans nest into a tree.

### Propagation

The trace context travels in **headers** (HTTP, gRPC, Kafka messages).

W3C standard:

```
traceparent: 00-<trace-id>-<span-id>-01
tracestate: vendor-specific
```

Every service-to-service call must propagate or context is lost. Most SDKs do this automatically.

### Instrumentation

**Automatic**:
- HTTP middleware wraps incoming requests; starts a server span.
- HTTP/gRPC client wraps outgoing requests; starts a client span as child of current.
- DB driver instrumented; adds a query span.
- Message-queue producer adds trace context to header; consumer continues.

**Manual**:
```python
with tracer.start_as_current_span("compute_recommendations") as span:
    span.set_attribute("user.id", user_id)
    span.set_attribute("candidate_count", len(candidates))
    result = score(candidates)
```

Don't go crazy. Auto-instrument the framework + RPC; manually add only what's interesting.

### Storage and Display

Spans are sent to a **collector** (OTel Collector) and stored in a backend (Tempo, Jaeger, Datadog APM, X-Ray, Honeycomb).

Backend stores by trace_id; query gives:

```
Timeline view:
0ms ─┬─ gateway (200ms)
     │
     ├─ 5ms: orders-service (180ms)
     │   │
     │   ├─ 6ms: db query (15ms)
     │   └─ 22ms: payments call (140ms)
     │       │
     │       └─ 30ms: stripe API (130ms)
     │
     └─ 195ms: emit event (5ms)
```

Visualizing a trace is half the value — "the request spent 130ms in Stripe."

### Sampling

Trace collection is expensive. Sample.

**Head-based**:
- Decide at request entry: keep or drop.
- Random (e.g., 1%).
- Deterministic (e.g., based on hash(trace_id)).
- Cheap; might miss rare critical traces.

**Tail-based**:
- Collect all spans into a buffer.
- After trace completes, decide based on status / latency.
- Keep all error traces; sample successful.
- Expensive (buffer); smarter.

Production: head + tail combined. Sample 1-5% as baseline; tail-include errors + slow.

### Trace-to-Log Correlation

Every log line includes the current trace_id:

```
2025-... [INFO]  service=orders trace_id=abc123 "starting payment"
2025-... [ERROR] service=payments trace_id=abc123 "card declined"
```

In log UI: click trace_id → see the full trace. In trace UI: click span → see logs.

Mature tools (Datadog, Honeycomb, Grafana) make this seamless.

### What to Add as Attributes

Useful:
- `http.method`, `http.status_code`.
- `db.system`, `db.statement` (truncated; no PII).
- `messaging.system`, `messaging.destination`.
- `user.id` (low cardinality if dashboards; otherwise span-only).
- `feature.flag.value` (test how flags affect performance).

Avoid:
- Big payloads (truncate or skip).
- Sensitive data (PII, secrets, payment info).

### Performance Overhead

- Auto-instrumentation: 1–5% CPU overhead typical.
- Span emission: small bytes/request; batched by collector.

For high-QPS services: heavy sampling (0.1%) + tail-sampling errors.

### Errors in Traces

Mark span as error on exception:

```python
span.set_status(StatusCode.ERROR)
span.record_exception(e)
```

Error spans get red in the UI. Searching "slow + error" gets you the worst spans fast.

### Common Mistakes

| Mistake | Fix |
|---------|-----|
| No trace context propagation | Each service sees a different trace ID; can't link |
| 100% sampling on a 100k-QPS service | Trace storage + collector overwhelmed |
| Adding `user_email` to every span | Cardinality + PII |
| Trace ID in URLs | Leaks |
| No correlation between traces and logs | Half the value lost |

### Common Tools

| Tool | Notes |
|------|-------|
| **Jaeger** | Open source; mature |
| **Tempo** | Grafana's; cheap S3-backed |
| **Honeycomb** | Best UX for query-style debugging |
| **Datadog APM** | Full-stack; expensive |
| **AWS X-Ray** | Native to AWS |
| **OpenTelemetry Collector** | Vendor-neutral pipeline |

For new systems: instrument with OTel SDK; pick a backend.

### Trace-Driven Investigation Loop

1. Alert fires: p99 latency above SLO.
2. Find slow trace examples (search by p99 percentile or status=error).
3. Examine the trace timeline → identify the slow span.
4. Open logs for that span (correlated by trace_id) → find the exact failure.
5. Fix and verify on traces.

This is the modern debugging workflow. Far better than `grep` over logs.

> [!NOTE]
> Distributed tracing turns "the system is slow" from a mystery into a diagram. Implementing it requires consistent instrumentation across services — invest once, save thousands of debugging hours.

### Interview Follow-ups

- *"How does sampling preserve rare events?"* — Tail sampling: keep all errors regardless of sample rate; combined with reservoir sampling for slow latencies.
- *"What's the difference between a span and an event?"* — Span has start + end (a duration). Event is a point-in-time annotation within a span.
- *"How does tracing work with async / batch processing?"* — Propagate trace context in message headers; consumer creates a span linked to the producer's span via `links` (cross-process causal relationship).
