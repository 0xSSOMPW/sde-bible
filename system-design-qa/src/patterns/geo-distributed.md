# Q: Designing geo-distributed systems — patterns for multi-region.

**Answer:**

Going from one region to many trades **latency** (closer to users) for **complexity** (distributed state, replication, conflict resolution). Most services start single-region and only multi-region when justified.

### Why Multi-Region

- **Lower latency** for users worldwide (TCP RTT vs same continent).
- **Disaster recovery** (regional outages).
- **Data residency** (EU users' data must stay in EU).
- **Compliance** (HIPAA, FedRAMP).

Cost: 2–5× infra, dramatically more engineering complexity.

### The Four Common Topologies

#### 1. Single Region, Multi-AZ

Most apps. AZs are isolated within one region; survive single-AZ outage.

Latency: ~ms within region.

Capacity: limited by region's total.

### 2. Active-Passive (DR)

```
Region A (primary)         Region B (standby)
   ▲                              ▲
   │ all traffic                  │
client                          (hot standby, cold standby)
```

All traffic hits primary. Standby kept up-to-date via async replication.

On primary failure: DNS / Anycast cuts over.

RTO: minutes (manual or automated).
RPO: replication lag (seconds typical).

#### 3. Active-Active (Geo-Distributed)

```
Region A           Region B           Region C
   ▲                  ▲                  ▲
   │ traffic from     │ traffic from     │ traffic from
   │ Americas         │ Europe           │ Asia
client            client             client
```

Each region serves nearby users. Data replicated across regions.

Key challenge: **cross-region writes**.

- Sync replication: cross-region RTT (80–150 ms) per write. Often too slow.
- Async replication: writes succeed locally, replicate later. Risk of conflicts.

#### 4. Region-Partitioned

User data is partitioned by region; each user has a "home region."

```
US users    → US cluster (all their data)
EU users    → EU cluster
APAC users  → APAC cluster
```

No cross-region writes for typical traffic. Travelers may hit a far region for their own data (longer latency).

Used by: Slack, Stripe, Cloudflare (partial), banks (mandated by regulation).

### Routing Users

**GeoDNS**: DNS server returns nearest region's IP based on resolver IP.
- Coarse; works at country level.
- Caches limit failover speed.

**Anycast**: same IP advertised from multiple PoPs via BGP. Internet routes to nearest.
- Sub-second failover.
- Used by Cloudflare, AWS Global Accelerator, Google.

**Client-side**: app pings several regions, picks fastest.
- Most accurate.
- Slow first request.

Production: layered. Anycast for L4 entry; app routes to nearest data home.

### Data Replication Patterns

| Pattern | Latency | Consistency | Use |
|---------|---------|-------------|-----|
| Sync cross-region (Spanner) | High write | Strongest | Financial, must-be-consistent |
| Async (Cassandra cross-DC, Postgres logical) | Local-fast | Eventual | Most |
| Multi-leader (e.g., DynamoDB Global Tables) | Local-fast | LWW | Independent regional state |
| Region-partitioned + no replication | Local-fast | Strong per partition | Per-user data with home region |

### Conflict Resolution

For multi-leader / active-active:
- **LWW (last-write-wins)**: simple, lossy.
- **App-level merge**: define per type (union for sets, max for counters).
- **CRDTs**: mathematically merge-safe.
- **Vector clocks**: detect concurrent writes; punt to app.

LWW is fine for some workloads (e.g., session prefs), wrong for others (financial).

### What Data Goes Where

Categorize your domain:

| Data class | Where |
|-----------|-------|
| Per-user (profile, history) | User's home region; replicate elsewhere if user travels |
| Global lookup (product catalog) | Replicated globally; eventual is fine |
| Truly global ledger (payments) | Distributed SQL (Spanner) or single primary with regional read replicas |
| Compliance-bound (EU data) | Pinned to region by law |
| Operational telemetry | Per-region, aggregated centrally |

### Failover and Disaster Recovery

For active-passive:
- Replication lag monitored.
- Quarterly failover drills.
- Documented runbook.
- DNS TTL ≤ 60 seconds.
- Standby region kept warm.

For active-active:
- Each region capacity-sized to absorb peer load (e.g., 3 regions, each sized for ~50% of total).
- Traffic routing must remove failed region quickly.
- Watch for ingress storms when traffic shifts.

### Data Residency

For EU users:
- Personal data stored in EU region.
- Audit trail of where data flowed.
- Sub-processors disclosed.

GDPR enforces this; HIPAA, China cybersecurity law, India DPDP do similar things.

Technical: tag data with origin; route storage / processing to correct region; ensure backups don't leak.

### Multi-Region Cost

- Compute: 2-3× single region.
- Egress between regions: very expensive ($0.02-$0.09/GB).
- Storage: replicated → 2-3× storage cost.
- Engineering: arguably the biggest line.

Don't go multi-region without a clear ROI.

### Common Mistakes

| Mistake | Reality |
|---------|---------|
| Sync replication across continents | Write latency dominates SLA |
| LWW on multi-region for financial data | Silent data loss |
| Two regions with no failover plan | Disaster doesn't follow your schedule |
| Same database backing both regions over WAN | DB latency kills app |
| Forgetting compliance | Big-deal lawsuits, not just bugs |

> [!NOTE]
> Multi-region is one of the highest-ROI levers for the right workload and one of the most expensive distractions for the wrong one. Justify with users, latency, compliance, or DR; don't do it because "it sounds modern."

### Interview Follow-ups

- *"How does Spanner support multi-region writes?"* — Each "split" (range) is a Paxos group spanning regions; writes commit when majority acks. TrueTime ensures external consistency.
- *"How would you handle a user who moves from US to EU?"* — Migrate their data to new home region; route from then on. Edge case; usually rare enough to script.
- *"What is 'follow-the-sun'?"* — Different regions active at different times of day; route to wherever it's daytime. Old pattern; modern systems mostly all-active.
