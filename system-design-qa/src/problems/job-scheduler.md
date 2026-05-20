# Q: Design a distributed job scheduler (cron at scale).

**Answer:**

Run scheduled or delayed jobs at scale across many workers. Tests **leader election**, **idempotency**, **at-least-once execution**, **time skew**, and **scaling out**.

### Requirements

**Functional**:
- Schedule a job: now, at a future time, or on a cron schedule.
- Cancel a scheduled job.
- Retry on failure with policy.
- Run jobs across many workers.

**Non-functional**:
- 10M scheduled jobs.
- 10k+ jobs/sec firing.
- p99 fire-time accuracy < 1 second.
- Survive node failure with no missed jobs.

### High-Level Architecture

```
Producer ──► Scheduler API ──► Job Store (Postgres / DDB)
                                    │
                                    ▼
                           Dispatcher (one or more)
                                    │
                                    ▼
                              Kafka / SQS
                                    │
                                    ▼
                           Worker pool
                                    │
                                    ▼
                          Result + retry logic
```

### Job Store

```sql
CREATE TABLE jobs (
    id              UUID PRIMARY KEY,
    schedule_type   ENUM ('once', 'cron'),
    fire_at         TIMESTAMPTZ,         -- next fire time
    cron_expr       TEXT,
    payload         JSONB,
    status          ENUM ('pending', 'in_progress', 'completed', 'failed'),
    retries         INT DEFAULT 0,
    max_retries     INT DEFAULT 3,
    locked_by       TEXT,
    locked_until    TIMESTAMPTZ
);

CREATE INDEX ON jobs (fire_at) WHERE status = 'pending';
```

Index on `fire_at` for the polling query.

### Dispatcher Loop

```
loop:
    fired = SELECT * FROM jobs
            WHERE status = 'pending' AND fire_at <= now()
            ORDER BY fire_at
            LIMIT 1000
            FOR UPDATE SKIP LOCKED;

    for j in fired:
        publish j to Kafka topic
        UPDATE jobs SET status='in_progress', locked_by=worker_id, locked_until=now()+5min
        WHERE id = j.id;
    sleep 100ms;
```

`FOR UPDATE SKIP LOCKED` (Postgres ≥ 9.5) lets many dispatcher instances run in parallel without conflicts.

### Worker Loop

```
on message from Kafka:
    try:
        execute(payload)
        UPDATE jobs SET status='completed' WHERE id = j.id;
    except RetryableError:
        retries += 1
        if retries < max_retries:
            UPDATE jobs SET status='pending', fire_at=now()+backoff, retries=retries
        else:
            UPDATE jobs SET status='failed'
            publish to DLQ
```

### Idempotency

Jobs may run more than once (at-least-once delivery). Workers must dedup:

```
on execute(job):
    INSERT INTO job_log (job_id, attempt) ON CONFLICT DO NOTHING
    if conflict: skip
    else: do the work
```

Or design the work itself to be idempotent (which is best).

### Cron Schedules

For recurring jobs, after firing:

```
UPDATE jobs
   SET fire_at = next_fire_time(cron_expr, now()),
       status  = 'pending',
       retries = 0
WHERE id = j.id;
```

Compute next fire using a cron library; advance `fire_at`.

### Scaling

**Vertical**: one dispatcher polls a Postgres table — easy to ~10k jobs/sec.

**Horizontal**: multiple dispatchers, sharded by job ID hash. Or by region/tenant.

**Tiered**:
- Short delays (< 1 hour) → Redis sorted set (ZSET) with score = fire_at; dispatcher pops due jobs.
- Medium / long → DB-backed table as above.

### Time Skew

Clocks drift. With NTP, ~50 ms typical. For tighter accuracy:

- One leader dispatcher (via etcd lease).
- Or: tolerate ~1 second jitter; for sub-second precision, use a dedicated time service.

### Cancellation

```
UPDATE jobs SET status='cancelled' WHERE id = ?;
```

Race: dispatcher may have already picked it up. Worker checks status before executing:

```
on execute:
    SELECT status FROM jobs WHERE id = ?;
    if cancelled: skip
```

Or use cancellation token in payload that the work itself respects.

### Retry / Backoff

Configurable per job type. Default: exponential with jitter.

```
retries=0  → backoff 30s
retries=1  → backoff 2min + jitter
retries=2  → backoff 10min + jitter
retries=3  → DLQ
```

### Failure Modes

| Failure | Handling |
|---------|---------|
| Dispatcher crashes mid-publish | Job stays `in_progress` until `locked_until` expires; re-dispatched |
| Worker crashes mid-execution | Same; idempotent execution avoids issues |
| Kafka unavailable | Dispatcher pauses; jobs accumulate in DB; resume after recovery |
| Job creates infinite loop (work spawns more jobs) | Rate-limit job creation; quota per tenant |
| Clock skew on worker | Use server time, not worker time |

### Observability

- `pending_jobs_count` by fire_at bucket.
- `firing_lag` (fire_at vs actual execution).
- `dispatch_latency_p99`.
- Failed jobs in DLQ.

Alert if `pending_jobs > N` ahead of schedule (dispatcher behind).

### Storage Picks

- **Postgres**: jobs table; ACID; row locking via `FOR UPDATE SKIP LOCKED`.
- **Redis ZSET**: short-delay jobs (timer wheel pattern).
- **Kafka**: dispatch / DLQ topics.
- **DynamoDB**: at extreme scale, sharded by `fire_at` bucket.

### Variants

**Workflow scheduler** (Temporal, Cadence, Step Functions):
- Stateful workflows with retries, sleep, signals.
- Built on a job scheduler at the bottom.
- Right answer when "scheduled job" → "5-step workflow with conditional branches."

**Cron service** (Kubernetes CronJob):
- For job *types* (one schedule, many runs over time).
- Less granular than per-event schedulers.

### Common Mistakes

| Mistake | Fix |
|---------|-----|
| Single dispatcher, no leader election | SPOF; use Raft / lease |
| Not handling `in_progress` job after crash | Stuck forever; expire `locked_until` |
| Workers fetch from DB directly | DB contention; use queue between dispatcher and workers |
| No idempotency | At-least-once dupes data |
| Retry without backoff | Hammers downstream during outages |

> [!NOTE]
> Cron at scale is two things: a **fast index** on "fire_at" and **at-least-once delivery to workers** with idempotency. Get those right; everything else is plumbing.

### Interview Follow-ups

- *"How do you handle 1M jobs firing at midnight?"* — Spread fires with random jitter; tiered dispatchers; pre-load due jobs ahead of time.
- *"How do you support 'run every minute on Tuesdays at 3PM Pacific'?"* — Cron expression library + per-job timezone.
- *"What if a job takes 2 hours?"* — Long-running jobs use heartbeats to extend `locked_until`; if heartbeat stops, redispatch.
