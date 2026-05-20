# Q: Strangler fig pattern — migrating from a legacy system without a big rewrite.

**Answer:**

Named after the tropical fig that grows around a host tree until the host dies. The pattern: a new system grows around the old one, gradually taking over functionality, until the old is shut down.

Beats the alternative ("big rewrite") which famously fails.

### When You Need It

- Monolith you can't replace in one go.
- Critical legacy system with tribal knowledge in code.
- Multi-year migration where the old must keep running.
- Re-platforming (on-prem → cloud, mainframe → microservices).

### The Pattern

```
year 0:                year 1:                  year 2:
                          
  [Monolith]           [Monolith]              [Monolith]
                          │ ──── shrinking ──── │
   handles all         [Façade]                [Façade]
   requests            ──┬──┐                   ──┬──┐
                         ▼  ▼                       ▼  ▼
                      [new]                       [new] [new] [new]
                                                  most traffic
                          
                                                 year 3: monolith retired
```

A **façade** sits in front of the legacy system. Some requests still go to the legacy; new functionality lives in new services. Over time, more is rerouted.

### The Façade

Typically a reverse proxy with routing rules:

```
GET /api/users/*       → new user-service
GET /api/orders/*      → legacy (still)
POST /api/checkout     → legacy
GET /api/dashboard     → new dashboard-service
```

As features migrate, edit the routing. Clients don't change.

### Approach

**1. Set up the façade.**

Edge proxy (Envoy, nginx, AWS ALB) in front. Initially routes everything to legacy.

**2. Pick a slice.**

Smallest functional piece you can extract. Often:
- Read-only endpoints first.
- Self-contained domains (auth, search).
- Recently-built modules (cleaner separation).

**3. Build new alongside.**

New service implements that slice. Tested independently.

**4. Shadow / dual-run.**

Send traffic to both old and new; compare responses. Flag discrepancies.

**5. Cut over.**

Switch façade rules; small percentage first (canary).

**6. Repeat.**

Each slice migrates the same way. Old system shrinks.

**7. Decommission.**

When legacy serves no traffic, archive it.

### Data Migration

The hardest part is data, not code.

Options:

**1. New writes go to new system; reads from both.**
- New starts empty; backfilled from old.
- Reads union; writes only to new for migrated keys.

**2. Dual-write.**
- Both systems get the write.
- Risk: divergence on partial failure.
- Add reconciliation jobs.

**3. CDC from legacy.**
- New system subscribes to legacy's change stream.
- Eventually consistent; near-zero divergence.
- Best pattern for large migrations.

**4. Old as system of record, new as cache/view.**
- Legacy is canonical; new builds a read-optimized projection.
- Until new is canonical, can revert easily.

### When to Cut Over

Per slice, gate criteria:
- New service hits same response shapes as old.
- Shadow comparison error rate < threshold.
- Functional + load tests pass.
- Runbook for rollback documented.

Then ramp: 1% → 5% → 25% → 50% → 100%.

### What Goes Wrong

| Problem | Mitigation |
|---------|-----------|
| Data drift between old and new | Reconciliation jobs, schema validation |
| New service depends on legacy internals | Refactor legacy to expose stable API first |
| Migration never finishes (zombie tail) | Force timeline; allocate migration team |
| New service has different bugs than old | Shadow comparison + canary |
| Org politics: who owns the migrated piece | Define ownership upfront |

### Strangler vs Big Rewrite

| Aspect | Strangler | Big Rewrite |
|--------|-----------|-------------|
| Risk | Low (incremental) | High (deploy-day binary cutover) |
| Time | Long (months/years) | Long (often unfinished) |
| Value delivery | Continuous | At the end |
| Org tolerance | Sustained focus | Often loses funding mid-way |
| Success rate | High | Famously low (Joel Spolsky's "Things You Should Never Do") |

### Patterns That Pair Well

- **Service mesh**: enables traffic shifting + canaries with no app changes.
- **Feature flags**: toggle between old/new code paths per user.
- **CDC**: pipe data between systems during migration.
- **API gateway**: hosts the façade routing.

### Anti-Patterns

| Anti-pattern | Reality |
|--------------|---------|
| "Rewrite everything in 6 months" | Misses scope by 5×; delivers nothing |
| New system without compatibility layer | Clients break |
| Migrate everything from one shared DB | New microservice tied to old schema; no real isolation |
| No measurement of what's still on legacy | Don't know when done |

> [!NOTE]
> Strangler fig is a discipline, not a tool. The key is incremental delivery of working code — never letting the new system get more than weeks ahead of being deployable.

### Interview Follow-ups

- *"How do you know when to start strangling vs when to add to the monolith?"* — Strangle when new features in the monolith block deployment of other features. Or when scale forces extraction.
- *"What if old and new have different consistency models?"* — Migration period accepts dual consistency; reconcile end-of-day; switch reads first to test, then writes.
- *"How do you keep team morale during a 3-year migration?"* — Visible progress per quarter; celebrate each slice retired; rotate engineers through the migration team.
