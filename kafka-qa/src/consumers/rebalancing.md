# Q: What is Consumer Rebalancing and why can it be problematic?

**Answer:**

A **rebalance** is the process of redistributing partition assignments among consumers in a group. It's triggered when the group membership changes.

### What Triggers a Rebalance?
1. A consumer **joins** the group (new instance deployed).
2. A consumer **leaves** the group (instance crashes or shuts down).
3. A consumer **fails to send a heartbeat** within `session.timeout.ms`.
4. A consumer's `poll()` calls take longer than `max.poll.interval.ms`.
5. **Partitions are added** to the subscribed topic.

### Why Rebalancing is Problematic

During a rebalance, **all consumers in the group stop processing**. This causes a processing pause (sometimes called "stop the world") that can last from milliseconds to minutes depending on the group size.

```
Normal operation:
  Consumer A ← P0, P1  (processing)
  Consumer B ← P2, P3  (processing)

Consumer B crashes → REBALANCE triggered:
  All consumers STOP processing
  Coordinator reassigns partitions
  Consumer A ← P0, P1, P2, P3 (resumes)
  
Total pause: seconds to minutes
```

### Rebalance Strategies

**1. Eager Rebalancing (Default in older versions)**
All consumers give up ALL partition assignments, then get new ones. Maximum disruption.

**2. Cooperative (Incremental) Rebalancing (Kafka ≥ 2.4)**
Only the partitions that need to move are revoked and reassigned. Other consumers continue processing without interruption.

```properties
partition.assignment.strategy=org.apache.kafka.clients.consumer.CooperativeStickyAssignor
```

```
Consumer B crashes:
  Consumer A: keeps P0, P1 (no pause!) + receives P2, P3
  Only P2, P3 are "moved" — minimal disruption
```

### Avoiding Unnecessary Rebalances

**Tune these consumer configs:**
```properties
# Time before a consumer is considered dead (default: 45s)
session.timeout.ms=45000

# Heartbeat interval (should be ~1/3 of session.timeout)
heartbeat.interval.ms=15000

# Max time between poll() calls before consumer is evicted
max.poll.interval.ms=300000   # 5 minutes

# Reduce records per poll if processing is slow
max.poll.records=500
```

> [!CAUTION]
> The most common cause of unnecessary rebalances is slow message processing. If processing a batch of messages takes longer than `max.poll.interval.ms`, Kafka assumes the consumer is dead and triggers a rebalance — even though it's still alive and processing. Either speed up processing, reduce `max.poll.records`, or increase `max.poll.interval.ms`.

### Static Group Membership (Kafka ≥ 2.3)
Assign a fixed identity to each consumer using `group.instance.id`. When a consumer restarts, it rejoins with the same identity and gets its previous partitions back — **no rebalance triggered** during brief restarts.

```properties
group.instance.id=consumer-host-1
```
