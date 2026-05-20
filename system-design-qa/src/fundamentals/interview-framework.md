# Q: How do you approach a system design interview — the structured framework.

**Answer:**

System design interviews are conversational, not technical trivia. The interviewer is testing whether you can drive a 40-minute design conversation, make justified trade-offs, and surface risks. The framework below is what most senior interviewers expect — adapt it, don't recite it.

### The 6-Step Framework

```
1. Clarify requirements      (5 min)
2. Estimate scale            (3 min)
3. API + data model          (5 min)
4. High-level design         (10 min)
5. Deep-dive on 2–3 areas    (15 min)
6. Trade-offs & extensions   (5 min)
```

Adjust the time per signal from the interviewer — if they push deeper on storage, spend more time on storage.

### 1. Clarify Requirements

Don't start drawing boxes. Ask:

**Functional**:
- Core features (pick 2–4; defer the rest).
- Read patterns vs write patterns.
- Multi-tenant? Multi-region?

**Non-functional**:
- DAU / scale.
- Latency target.
- Consistency requirements (per data class).
- Availability target (SLO).
- Mobile vs web? Geographic distribution?

Verbalize: *"I'm assuming we don't need to design the mobile client, just the backend. Sound right?"*

This step is graded. Skipping it is a red flag.

### 2. Estimate Scale

Numbers ground the design. See [Back-of-Envelope Estimation](./back-of-envelope.md).

Compute:
- QPS (read and write separately).
- Storage growth (per day, per year).
- Bandwidth (peak egress).

Match against known limits:
- > 50k QPS → cache + replicas.
- > 10 TB → sharding.
- > 1 Gbps egress → CDN.

### 3. API + Data Model

**API first** — what's the contract?

```
POST   /v1/orders         { items: [...], userId: ... }    → 201 { orderId }
GET    /v1/orders/{id}                                      → 200 { order }
GET    /v1/orders?userId=&cursor=                           → 200 { orders, nextCursor }
PATCH  /v1/orders/{id}    { status: "cancelled" }           → 200
```

State idempotency: "I'd require `Idempotency-Key` for POST."

**Then the data model** — pick a schema and a storage class.

```
Orders Table (Postgres, sharded by userId)
  id            UUID PK
  user_id       BIGINT  (shard key)
  status        ENUM
  amount        NUMERIC
  created_at    TIMESTAMPTZ
  INDEX (user_id, created_at DESC)
```

For NoSQL choices: state partition key + sort key + access pattern.

### 4. High-Level Design

Draw boxes. Standard layers:

```
Client → CDN → Load Balancer → API Gateway → Service(s) → Cache → DB
                                                       ↓
                                                    Queue → Async Workers
```

Annotate what each layer does. Common ones:

- **CDN**: static assets, media.
- **LB**: L7 (Envoy/HAProxy/Nginx) for HTTP routing.
- **API Gateway**: AuthN, rate limit, request shaping.
- **Stateless services**: business logic.
- **Cache**: Redis for hot reads.
- **Primary DB**: source of truth.
- **Search index**: Elastic / OpenSearch.
- **Queue/Stream**: Kafka / SQS for async.

Tell a request's story end to end. "User clicks 'Order Now' → API gateway → orders service → write to Postgres → publish event to Kafka → notify service → SMS."

### 5. Deep Dive on 2–3 Areas

Pick the two or three areas where the interview signal is highest. Common deep-dive prompts:

- **Sharding**: which key, how to rebalance, hot shards.
- **Caching**: invalidation strategy, eviction, stampede protection.
- **Consistency**: which operations need linearizability vs eventual.
- **Failure handling**: what happens if X dies — service, DB primary, region.
- **Hot keys**: how the system handles a celebrity user / viral post.
- **Scaling**: where the next bottleneck is and how you'd address it.

Walk through one in detail; let the interviewer steer the others.

### 6. Trade-offs & Extensions

Self-critique:

- "I'd start without sharding; add it once we cross 5k write QPS."
- "Cache is a write-through pattern; trades latency on writes for stronger reads."
- "We'd lose data in a multi-region disaster; for orders, switch to synchronous cross-region writes once revenue justifies the latency hit."

Mention what you'd add given more time: search, recommendations, monitoring, on-call playbooks.

### Anti-Patterns

| Anti-pattern | What to do instead |
|--------------|-------------------|
| Jumping to boxes before clarifying | Spend 5 minutes on requirements |
| Reciting buzzwords (Kafka! Kubernetes! CRDTs!) | Justify every component you add |
| Overengineering for hypothetical scale | "We start simple; here's the migration path" |
| Ignoring failure modes | Always have an answer for "what if X dies?" |
| Drawing 30 boxes with no labels | Fewer, more thoughtful components |
| Picking technologies you don't understand | Stick to what you can defend |

### Signals Interviewers Look For

**Senior signals**:
- Asks clarifying questions before designing.
- Estimates before optimizing.
- Picks trade-offs deliberately, not by default.
- Names the bottleneck before being asked.
- Talks about operability (deploys, alerts, on-call).
- Pushes back on under-specified asks.

**Junior signals**:
- Knows specific tech stacks.
- Can draw a typical architecture.

**Red flags**:
- "We need it consistent, so let's use [strong-consistency DB]" for everything.
- Designing the perfect 100M-user system on minute one.
- Never mentioning failure, observability, or cost.

### Useful Phrases

- "For now, let's assume X. We can revisit."
- "I see two options: A trades latency for consistency, B does the opposite. I'd pick A because..."
- "The next bottleneck would be the DB write throughput. To go further: shard by user_id."
- "If a region goes down, here's what happens..."
- "I'd capture this metric and alert on it."

### Common Mistakes

| Mistake | Fix |
|---------|-----|
| One-line answers ("just use Cassandra") | Justify with the constraint that drove it |
| Treating it as a pop quiz | It's a conversation; ask, respond, iterate |
| Avoiding numbers | Drive every decision with an estimate |
| Forgetting non-functional requirements | Latency, availability, cost matter as much as features |
| Designing for the average request | Tail latency, hot keys, abuse — the interesting cases |

> [!NOTE]
> Best preparation: design 10–20 real systems out loud, in 45 minutes each, against a friend or alone. The framework becomes automatic; you spend the interview thinking about trade-offs, not the structure.

### Interview Follow-ups

- *"What if I gave you another hour?"* — Always have a list: observability, capacity planning, deployment story, security threats, cost optimization.
- *"What would you NOT do here?"* — Show judgment: "I wouldn't use a graph DB for this; the query pattern is key-value."
- *"How would you migrate from the existing system?"* — Strangler fig pattern; dual writes; feature flag the cutover.
