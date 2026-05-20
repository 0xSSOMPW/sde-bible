# Q: What are SLI, SLO, SLA, and error budgets — and how do they connect?

**Answer:**

SRE terminology that often confuses engineers. The three are nested:

- **SLI**: a measurement.
- **SLO**: an internal target for that measurement.
- **SLA**: a contract with consequences if you miss the SLO.
- **Error budget**: how much you can afford to miss the SLO over a window.

### SLI — Service Level Indicator

A quantitative measure of one aspect of service quality. Common SLIs:

- **Availability**: ratio of successful responses to total.
  ```
  successful_requests / total_requests
  ```
- **Latency**: fraction of requests faster than threshold.
  ```
  count(requests where duration < 200ms) / total_requests
  ```
- **Throughput**: requests per second.
- **Correctness**: ratio of responses that match a golden expectation (rare, valuable).
- **Durability**: probability of data loss over time (mostly for storage).
- **Freshness**: how stale the data behind a request can be.

### SLO — Service Level Objective

A target for an SLI over a window:

```
99.9% of requests succeed, measured over 30 days
99% of p99 latencies ≤ 200 ms, measured over 7 days
```

The "9s" budget mapped to downtime:

| SLO | Allowed downtime per 30 days |
|-----|------------------------------|
| 99% | 7h 12m |
| 99.5% | 3h 36m |
| 99.9% | 43m |
| 99.95% | 21m |
| 99.99% | 4m 19s |
| 99.999% ("five nines") | 26 seconds |

Anything past 99.99% requires multi-region active-active and disciplined release engineering.

### SLA — Service Level Agreement

A *contract* between you and a customer, often with refund clauses:

```
"We guarantee 99.9% monthly uptime. Below that, you get 10% credit.
 Below 99% you get 25% credit."
```

The SLO is *internal* (your engineering target). The SLA is *external* (legal). The SLO should be **tighter** than the SLA — leave headroom for unknowns and customer-friendly fudge.

### Error Budget

If SLO = 99.9% availability over 30 days, the **error budget** is the 0.1% — about 43 minutes of allowed downtime per month.

Used for:

**1. Release velocity decisions.**
Budget remaining → ship aggressively.
Budget exhausted → halt risky changes; focus on reliability.

**2. Engineering investment.**
Frequent budget burn → pay down reliability debt.
Budget unused → invest in features instead.

```
                       error budget
   100% ────────────────────────────────  SLO ceiling
        │
        │                ╱╲
        │              ╱    ╲                ← burn
   99.9%├────────────╱────────╲────────────  SLO target
        │          ╱            ╲
        │
        │
   99.5%│
        └────────────────────────────────►  time
```

### Burn Rate Alerts

Better than threshold alerts: alert on **how fast** you're burning the budget.

```
Burn rate = (errors_in_window / requests_in_window) / (1 - SLO)

Burn rate = 1   → you'd consume the budget exactly on schedule
Burn rate = 14  → you'd burn 30 days' budget in ~2 days
```

Common dual-burn-rate alert (Google SRE Workbook):
- Fast: 14× burn rate over 1 hour AND 5 minutes → page.
- Slow: 6× burn rate over 6 hours AND 30 minutes → ticket.

Catches outages quickly without paging on every flap.

### Designing SLIs

Good SLI properties:

- **User-perceived**: measure what users feel, not what your dashboard fancies.
- **Computable**: from existing telemetry, no manual instrumentation needed.
- **Aggregable**: by service, by API, by tenant.
- **Cheap**: doesn't itself cost the budget to compute.

The two go-to SLIs for any HTTP service:

```
availability = sum(rate(http_requests{status!~"5.."}[5m]))
             / sum(rate(http_requests[5m]))

latency      = sum(rate(http_requests_bucket{le="0.2"}[5m]))
             / sum(rate(http_requests[5m]))
```

### Choosing SLO Targets

| Service type | Reasonable SLO |
|--------------|---------------|
| Internal tools | 99% |
| Customer-facing UI | 99.9% |
| Payment / financial APIs | 99.95% |
| Critical infra (DNS, auth) | 99.99% |

Higher targets force expensive architecture (multi-region, sync replication). Don't promise five-nines without buying the engineering.

### Decoupling Internal SLO from External SLA

```
SLA (external):    99.9%
SLO (internal):    99.95%        ← tighter, gives headroom
Budget alert at:   99.97%        ← act before SLO threatened
```

If you fire alerts at the SLA threshold you've already missed it. Bake in margin.

### Where Each Number Lives

```
SLO ┐
    ├── alert rules (burn-rate based)
    │
    ├── dashboard headline ("we're at 99.91% this week")
    │
    ├── feature freeze trigger (budget < 0)
    │
    └── post-mortem ROI ("how much SLO did this incident cost?")
```

### Common Mistakes

| Mistake | Fix |
|---------|-----|
| Identical SLA and SLO | Tighten SLO to leave headroom |
| SLOs for internal metrics nobody cares about (CPU usage) | Pick user-perceived measures |
| 100% SLO | Impossible; even AWS doesn't claim it |
| Alerting on every below-threshold blip | Use burn rate over windows |
| Treating SLO as a goal to overshoot | If you're consistently at 99.999% on a 99.9% SLO, you're over-investing — spend the budget on features |

> [!NOTE]
> Error budgets reframe reliability from "always more" to "exactly enough." A team that's never burning its budget should be shipping faster; a team always over budget should slow down and harden.

### Interview Follow-ups

- *"How do you decide SLO targets for a brand-new service?"* — Start with a lenient SLO (99.5%) for a few months, gather data, tighten once stable. Don't promise what you haven't measured.
- *"What if a dependency violates its SLO?"* — Either renegotiate yours, mitigate via fallback (cache last-known-good), or split usage to a more reliable dependency.
- *"What's the difference between MTTR, MTTD, and MTTF?"* — Mean Time To Recovery (how long an outage lasts), Detection (how long to notice), Failure (between failures). All three feed into SLO performance.
