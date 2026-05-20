# Q: Redundancy, failover, multi-AZ/region — designing for failure.

**Answer:**

Every component will eventually fail. Reliability comes from **redundancy** (more than one of each thing) + **failover** (automatically reroute when one fails). Cost rises non-linearly: single-AZ → multi-AZ → multi-region → globally active-active.

### Failure Domains

Anything that fails together is one **failure domain**:

- A single server.
- A rack.
- An availability zone (AZ).
- A region.
- A cloud provider.

Design so that one failure domain dying doesn't take down your service.

### Single AZ (Not Enough)

- One AZ → AWS or GCP outage in that AZ takes you down (happens yearly).
- Not redundant against ZA/networking issues.

### Multi-AZ

Spread across 2–3 AZs in the same region:

- Latency between AZs: ~1–2 ms.
- Cross-AZ bandwidth typically free or cheap.
- Survives single-AZ outage.

Standard for production:

```
LB (multi-AZ)
   │
   ├── app servers across AZ1, AZ2, AZ3
   ▼
DB primary in AZ1, replicas in AZ2, AZ3
```

### Multi-Region

For regional disasters or compliance (data residency):

- Latency cross-region: 50–150 ms.
- Egress between regions: expensive.
- Cross-region sync replication: latency penalty.

Patterns:

**1. Active-Passive (DR).**
Primary region serves all traffic. Standby is hot or warm. Failover triggers (manually or automatically) on regional outage.

RPO (data loss window) = replication lag.
RTO (recovery time) = how long to switch.

**2. Active-Active.**
Both regions serve traffic. Writes go to nearest. Replication is async.

Requires conflict resolution (multi-leader semantics). Best for partitioned data: each region owns its own user base.

**3. Hot Standby with Read Replicas.**
Reads in any region; writes only in primary. Higher write latency for distant users.

### Failover Mechanics

```
1. Detect: health checks fail consecutively (alert + auto-detect).
2. Verify: not a transient blip; confirm region is unreachable.
3. Promote standby DB to primary.
4. Update DNS / routing to point at standby.
5. Bring up app tier (or it was already warm).
6. Drain residual connections to old primary (if reachable).
```

Trade-off: aggressive failover causes flapping; conservative loses time.

### DNS-Based vs Anycast Routing

DNS-based:
- TTL determines failover speed (30–60s minimum).
- Cached by resolvers, ignores TTL sometimes.

Anycast:
- Same IP advertised from multiple PoPs via BGP.
- Network routes to topologically nearest.
- Failover is sub-second.
- Used by Cloudflare, AWS Global Accelerator.

### Data Replication Choices

| Mode | RPO | RTO | Cost |
|------|-----|-----|------|
| Sync replication cross-region | Zero | Seconds | Latency on every write |
| Semi-sync (one remote replica) | Tiny window | Seconds | Some latency |
| Async cross-region | Replication lag | Seconds | No write latency |

Sync cross-region is brutal for write latency (100+ ms each). Most use async + accept the small RPO.

### Cell-Based Architecture

Split users into **cells**, each a complete independent stack:

```
Cell 1: users [0–10M]    → own DB, app, cache
Cell 2: users [10M–20M]  → own DB, app, cache
...
```

Outage in Cell 5 doesn't affect Cell 3. Blast radius bounded.

Used by AWS, Stripe, Slack, Salesforce.

### Chaos Engineering

You can't claim reliability without testing it. **Chaos engineering**:

- Inject failures in production (Netflix Chaos Monkey).
- Game-day simulations: take down an AZ during business hours.
- Validate runbooks: practiced or theoretical?

If you've never tested failover, it doesn't work.

### Graceful Degradation

When dependencies fail, degrade rather than crash:

- Cache unavailable → fall back to DB (slower, still works).
- ML model service down → fall back to popular items.
- Email send fails → queue for retry.

See [Graceful Degradation](./graceful-degradation.md).

### Per-Layer Redundancy

| Layer | Approach |
|-------|---------|
| LB | Multi-AZ managed LB (ALB, NLB) |
| App tier | Multi-AZ, autoscaling, healthchecked |
| Cache | Redis Cluster across AZs; expect a portion of data lost on AZ loss |
| DB | Primary in one AZ, sync replica in another |
| Queue | Multi-AZ Kafka cluster |
| Storage | S3 (cross-AZ by default), Glacier for backup |
| CDN | Global by definition |

### Failover Tests (Game Days)

Periodically practice:
- Kill the primary DB; measure failover time.
- Take down an AZ at the network level.
- Force a region failover; measure data loss.
- Verify runbooks work end-to-end.

### Common Mistakes

| Mistake | Reality |
|---------|---------|
| "We have backups" = "we're highly available" | Backups protect against logical disasters; not the same as redundancy |
| 2 instances in one AZ | Both die when the AZ dies |
| No tested failover | "Will work in theory" |
| Manual failover with no runbook | Outage longer than necessary |
| Replication lag never alerted | Lossy failover surprises |

> [!NOTE]
> Reliability is a function of testing, not architecture diagrams. Until you've taken down each tier in production and seen the system recover, you don't know whether it works.

### Interview Follow-ups

- *"What's your RPO and RTO target?"* — Per service, per tier. Recovery time vs data loss is a budget you spend.
- *"How do you avoid cross-region sync writes for an active-active app?"* — Partition by user/region; each region writes its own partition; replicate async.
- *"How do you do cross-region capacity planning?"* — Each region sized to handle its share + N/(N-1) when peer regions are out, to absorb failover traffic.
