# Summary

- [Introduction](./README.md)

---

# Fundamentals

- [Core Concepts]()
  - [CAP & PACELC](./fundamentals/cap-pacelc.md)
  - [ACID vs BASE](./fundamentals/acid-vs-base.md)
  - [Consistency Models](./fundamentals/consistency-models.md)
  - [Latency, Throughput, Tail Latency](./fundamentals/latency-throughput.md)
  - [Back-of-Envelope Estimation](./fundamentals/back-of-envelope.md)
  - [Scalability: Vertical vs Horizontal](./fundamentals/scalability.md)
  - [SLO, SLI, SLA, Error Budgets](./fundamentals/slo-sli-sla.md)
  - [How to Approach a System Design Interview](./fundamentals/interview-framework.md)

---

# Building Blocks

- [Networking & Communication]()
  - [Load Balancers (L4 vs L7)](./building-blocks/load-balancers.md)
  - [CDN & Edge Caching](./building-blocks/cdn.md)
  - [API Styles: REST, gRPC, GraphQL](./building-blocks/api-styles.md)
  - [WebSockets, SSE, Long Polling](./building-blocks/realtime-communication.md)
  - [API Gateway & BFF](./building-blocks/api-gateway-bff.md)

- [Data Storage]()
  - [SQL vs NoSQL](./building-blocks/sql-vs-nosql.md)
  - [Database Indexing](./building-blocks/indexing.md)
  - [Sharding & Partitioning](./building-blocks/sharding-partitioning.md)
  - [Replication (Leader-Follower, Multi-Leader, Leaderless)](./building-blocks/replication.md)
  - [Object Storage (S3-like)](./building-blocks/object-storage.md)
  - [Time-Series Databases](./building-blocks/time-series-db.md)

- [Caching]()
  - [Caching Strategies](./building-blocks/caching-strategies.md)
  - [Cache Invalidation & Eviction](./building-blocks/cache-invalidation.md)
  - [Cache Stampede & Thundering Herd](./building-blocks/cache-stampede.md)
  - [Distributed Cache (Redis, Memcached)](./building-blocks/distributed-cache.md)

- [Messaging]()
  - [Queue vs Pub/Sub vs Log](./building-blocks/messaging-models.md)
  - [Delivery Semantics & Idempotency](./building-blocks/delivery-semantics.md)
  - [Backpressure](./building-blocks/backpressure.md)

---

# Distributed Systems

- [Consensus & Coordination]()
  - [Consensus: Raft & Paxos](./distributed/consensus.md)
  - [Leader Election](./distributed/leader-election.md)
  - [Distributed Locking](./distributed/distributed-locking.md)
  - [Two-Phase Commit & Beyond](./distributed/two-phase-commit.md)

- [Time, Order, Identity]()
  - [Clocks: Lamport, Vector, Hybrid Logical](./distributed/clocks.md)
  - [Unique ID Generation (Snowflake, UUID, ULID)](./distributed/id-generation.md)
  - [Gossip Protocols](./distributed/gossip.md)

---

# Reliability & Resilience

- [Failure Patterns]()
  - [Redundancy, Failover, Multi-AZ/Region](./reliability/redundancy.md)
  - [Retries, Backoff, Jitter](./reliability/retries-backoff.md)
  - [Circuit Breaker, Bulkhead, Timeout](./reliability/circuit-breaker-bulkhead.md)
  - [Idempotency & Exactly-Once Effects](./reliability/idempotency.md)
  - [Graceful Degradation & Load Shedding](./reliability/graceful-degradation.md)

---

# Architecture Patterns

- [Microservice & Data Patterns]()
  - [CQRS](./patterns/cqrs.md)
  - [Event Sourcing](./patterns/event-sourcing.md)
  - [Saga (Choreography vs Orchestration)](./patterns/saga.md)
  - [Transactional Outbox](./patterns/outbox.md)
  - [Change Data Capture (CDC)](./patterns/cdc.md)
  - [Sidecar & Service Mesh](./patterns/sidecar-mesh.md)
  - [Strangler Fig (Legacy Migration)](./patterns/strangler-fig.md)

- [Coordination & Scaling Patterns]()
  - [Rate Limiting Algorithms](./patterns/rate-limiting.md)
  - [Consistent Hashing](./patterns/consistent-hashing.md)
  - [Bloom Filters & Probabilistic Structures](./patterns/bloom-filters.md)
  - [Geo-Distributed Systems](./patterns/geo-distributed.md)

---

# Security

- [AuthN, AuthZ, Hardening]()
  - [OAuth2 & OIDC](./security/oauth-oidc.md)
  - [JWT vs Session Cookies](./security/jwt-vs-sessions.md)
  - [Encryption: At Rest, In Transit, End-to-End](./security/encryption.md)
  - [API Rate Limiting & DDoS Protection](./security/ddos-rate-limit.md)

---

# Observability

- [Telemetry]()
  - [Metrics, Logs, Traces (Three Pillars)](./observability/three-pillars.md)
  - [Distributed Tracing](./observability/distributed-tracing.md)

---

# Top Design Problems

- [Classic Problems]()
  - [Design a URL Shortener (bit.ly)](./problems/url-shortener.md)
  - [Design a Rate Limiter](./problems/rate-limiter.md)
  - [Design Twitter / News Feed](./problems/news-feed.md)
  - [Design a Chat System (WhatsApp/Slack)](./problems/chat-system.md)
  - [Design a Ride-Sharing Service (Uber)](./problems/ride-sharing.md)
  - [Design a Distributed Cache](./problems/distributed-cache-design.md)
  - [Design a Notification System](./problems/notification-system.md)
  - [Design a Search Autocomplete (Typeahead)](./problems/typeahead.md)
  - [Design a Video Streaming Service (YouTube/Netflix)](./problems/video-streaming.md)
  - [Design a Payment System (Stripe-like)](./problems/payment-system.md)
  - [Design an Ad-Click Aggregation Pipeline](./problems/ad-click-pipeline.md)
  - [Design a Recommendation System](./problems/recommendation-system.md)
  - [Design a File Storage Service (Dropbox/Google Drive)](./problems/file-storage.md)
  - [Design a Web Crawler](./problems/web-crawler.md)
  - [Design a Distributed Job Scheduler](./problems/job-scheduler.md)
  - [Design a Metrics / Monitoring System](./problems/metrics-system.md)
  - [Design a Distributed Counter](./problems/distributed-counter.md)
  - [Design a Geo-Proximity Service (Yelp/Nearby)](./problems/geo-proximity.md)
