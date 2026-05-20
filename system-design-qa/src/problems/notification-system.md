# Q: Design a notification system (push, email, SMS).

**Answer:**

Multi-channel delivery, deduplication, per-user preferences, rate limits, retries. The interview tests **fan-out**, **idempotency**, **third-party integration**, and **failure handling**.

### Requirements

**Functional**:
- Multiple channels: push (mobile), email, SMS, in-app.
- Per-user channel preferences.
- Templates with variable substitution.
- Scheduling (send now / send later).
- Deduplication (don't notify twice for one event).
- Aggregation ("you have 5 new likes" instead of 5 separate notifications).
- Delivery / open / click tracking.

**Non-functional**:
- 100M users.
- 10B notifications/day → ~120k/sec.
- < 1 minute end-to-end latency for "send now."
- Provider failures (APNs/FCM/Twilio) don't break flow.

### High-Level Architecture

```
Producer services (orders, social, security)
                    │
                    ▼
       Kafka topic "notifications"
                    │
                    ▼
        Notification Service
              ├── preference check
              ├── deduplication
              ├── template render
              ├── rate limit per user
              ▼
         Per-channel workers
                ├── Push (APNs/FCM)
                ├── Email (SES/Sendgrid)
                ├── SMS  (Twilio)
                └── In-app (WS push or store)
                    │
                    ▼
           Provider APIs
                    │
                    ▼
           Bounce / delivery tracker
```

### Notification Lifecycle

```
1. Producer emits event ("order_placed", user_id=42)
2. Notification service consumes:
   - Look up user.preferences[event_type]
   - Render template
   - Check user rate limit (max N notif / day)
   - Dedup by (user_id, event_id, channel)
   - Route to per-channel worker
3. Channel worker calls provider API
4. Provider acks; worker stores delivery state
5. Provider webhooks confirm delivered/opened/bounced
```

### Producer API

```
POST /v1/notify
{
  "event": "order_shipped",
  "user_id": "u42",
  "data": { "order_id": "o123", "tracking_url": "..." },
  "channels": ["push", "email"],         // optional override
  "dedup_key": "order_shipped_o123"      // idempotency
}
```

`dedup_key` prevents re-notification on event replay (essential when producers are at-least-once).

### Per-User Preferences

```
user_id | event_type        | channels        | quiet_hours
u42     | order_shipped     | push, email     | 22:00–07:00
u42     | promotional_email | (none)
```

Look up cheap: Postgres or DynamoDB by user_id. Cache hot users in Redis.

### Templates

```
templates table:
  template_id | channel | locale | subject | body
```

Render with variable substitution (Mustache / Handlebars). Pre-compile templates; A/B test versions; locale-aware.

### Deduplication

```sql
INSERT INTO notification_log (dedup_key, user_id, sent_at)
VALUES (?, ?, now())
ON CONFLICT (dedup_key) DO NOTHING
RETURNING ...
```

If insert returns nothing, skip — already sent.

`dedup_key` namespace: `<event_type>:<entity_id>:<user_id>:<channel>` typically.

### Aggregation

User receives 100 "your photo was liked" notifications in 10 minutes. Batch:

```
on like event:
  add to "pending:user:u42:photo_liked" set with timestamp
  schedule flush in 5 minutes (if not already scheduled)

on flush:
  if count > threshold:
    "X people liked your photo"
  else:
    individual notifications
```

Implementation: Redis sorted set per user-event-type with TTL; cron consumer.

### Rate Limiting (per user)

Hard limit: e.g., max 10 notifications/day. Anti-spam protection.

```
INCR rate:user:u42:daily
EXPIRE 86400
if value > 10: drop
```

Different limits per channel and per priority (critical security notifications bypass).

### Channel Workers

Each provider is a separate service; isolation prevents one provider's outage from cascading.

**Push (APNs / FCM)**:
- HTTP/2 connections, persistent.
- Token management: store APNs/FCM tokens per device per user.
- Stale tokens cleanup (on bounce).

**Email (SES / Sendgrid)**:
- Per-domain sending limits (reputation management).
- Bounce / complaint webhooks → mark user as undeliverable.

**SMS (Twilio)**:
- Cost per message; rate-limit aggressively.
- Country-specific regulations.

**In-app**:
- WebSocket push for online users.
- Persisted to DB for offline; fetched on open.

### Provider Failures

Each call to a provider can fail (5xx, timeout). Pattern:

```
retry with exponential backoff (up to 3–5 attempts)
if all fail → write to DLQ for manual / scheduled retry
```

Circuit breaker per provider — open if error rate exceeds threshold.

Fallback channel: if push fails 3 times, try email.

### Delivery Tracking

- Provider webhook → tracker service.
- Persist state machine: queued → sent → delivered → opened → clicked / bounced.
- Stream to analytics (ClickHouse) for dashboards.

### Scheduling

"Send tomorrow at 9 AM" notifications.

- Insert to `scheduled_notifications (id, fire_at, payload)`.
- Cron worker: `SELECT FOR UPDATE SKIP LOCKED WHERE fire_at < now() LIMIT 1000`.
- Enqueue to main pipeline.

Or use a job scheduler (Sidekiq, Temporal, AWS EventBridge Scheduler).

### Quiet Hours / Time Zone

User in NYC at 22:00 doesn't want a push. Solutions:
- Look up user TZ.
- Defer non-critical notifications until morning.
- Critical (security, payments) bypass.

### Multi-Region

- Each region runs the pipeline locally.
- Push tokens region-aware (Apple's CDN routes globally; FCM same).
- SMS provider may charge less from one region; route accordingly.

### Storage

| Data | Store | Why |
|------|-------|-----|
| Templates | Postgres | Versioned, cached in app |
| User preferences | DynamoDB | Per-user key lookup |
| Device tokens | DynamoDB | (user_id, device_id) → token |
| Notification log | Cassandra | Append-only, high write |
| Scheduled jobs | Postgres | ACID for "send once" |
| Delivery tracking | ClickHouse | Aggregations |

### Failure Modes

| Failure | Handling |
|---------|---------|
| Provider 5xx | Retry; circuit-break; fallback channel |
| Stale push token | Provider returns "Unregistered" → mark token dead |
| Producer floods | Per-event-type rate limit; backpressure |
| Template bug formats wrong | Canary releases; per-template error rate alert |
| Same notification sent twice | Dedup key insert with unique constraint |

### Common Mistakes

| Mistake | Fix |
|---------|-----|
| Sending notifications synchronously in HTTP handler | Async via queue |
| One template per provider per channel manually maintained | Template service with composition |
| No per-user rate limit | One bad producer spams users |
| No bounce handling | Send to dead emails forever, hurt sender reputation |
| Storing entire payload in queue | Use IDs + fetch from DB |

> [!NOTE]
> The defining challenge is **isolation**: one channel/provider/template/user must never affect others. Per-provider workers, per-user limits, per-tenant quotas — all layers of isolation.

### Interview Follow-ups

- *"How would you A/B test notification text?"* — Variant assignment on send; track delivery → open → action per variant; pick winner.
- *"How do you not wake users at 3 AM?"* — Per-user TZ; deferral queue for low-priority; bypass for critical.
- *"How do you handle a user with 100 devices?"* — Each device tokens are independent; fan-out per token; coalesce when one device acknowledges read.
