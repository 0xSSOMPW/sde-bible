# Q: What is Cooperative Rebalancing and how does KIP-848 change consumer groups?

**Answer:**

Rebalancing is how a consumer group redistributes partitions when membership changes (consumer joins/leaves, topic gains partitions). For years, Kafka used **eager** rebalancing, which had a stop-the-world problem. Cooperative rebalancing fixes that, and **KIP-848** rewrites the protocol entirely to push coordination off the clients.

### Eager Rebalancing (the old default)

```
t0: consumers A, B, C own partitions [p0..p8]
t1: consumer D joins
t2: ALL consumers REVOKE all partitions  <-- stop-the-world
t3: leader computes new assignment
t4: ALL consumers receive new assignment, resume
```

Every rebalance — even adding a single consumer — forced every member to drop all partitions and reprocess from the last committed offset. For a group consuming 200 partitions, that meant a multi-second processing gap on every scale event.

### Cooperative Rebalancing (KIP-429, default since 3.0)

Set via `partition.assignment.strategy=org.apache.kafka.clients.consumer.CooperativeStickyAssignor`.

```
t0: A=[p0,p1,p2], B=[p3,p4,p5], C=[p6,p7,p8]
t1: D joins
t2: Plan: D gets [p2, p5, p8]. A,B,C revoke ONLY those.
t3: A=[p0,p1], B=[p3,p4], C=[p6,p7]   <-- still consuming!
t4: D picks up [p2,p5,p8]
```

Key properties:
- **Incremental**: members keep partitions not being reassigned.
- **Two-phase**: first revoke just the moving partitions, then assign.
- **Sticky**: assignor tries to preserve previous ownership to minimize state warm-up (important for Kafka Streams).

### KIP-848: The Next Consumer Rebalance Protocol

Available as preview in 3.7+, GA in 4.0. It moves group coordination from a client-side leader to the **group coordinator broker**.

Old protocol problems:
- Group "leader" is one of the consumers — it must download metadata and compute assignments. Slow with many partitions.
- Heartbeats, sync, and join are tangled — slow members stall the whole group.
- Static membership and cooperative are bolt-ons.

KIP-848 changes:
- Broker computes assignments using a server-side assignor.
- Consumers send a single `ConsumerGroupHeartbeat` RPC — no JoinGroup/SyncGroup dance.
- Reconciliation is per-member and asynchronous — slow consumers don't block fast ones.
- Rebalance time drops dramatically for large groups (hundreds of consumers, thousands of partitions).

```
Old:                                    New (KIP-848):
JoinGroup → SyncGroup → Heartbeat       Heartbeat (carries everything)
  (synchronous, leader-driven)            (async, broker-driven)
```

### Comparison

| Aspect | Eager | Cooperative (KIP-429) | KIP-848 |
|--------|-------|----------------------|---------|
| Stop-the-world | Yes | No | No |
| Assignment computed by | Client leader | Client leader | Broker |
| Rebalance time (1000 partitions) | Seconds | Seconds (smaller) | Sub-second |
| Heartbeat handling | Tied to rebalance | Tied to rebalance | Decoupled |
| Static membership support | Bolt-on (KIP-345) | Yes | Native |

### Migration Caveats

You can't flip mid-flight from eager to cooperative without care. The supported path:

1. Roll consumers with `partition.assignment.strategy=[CooperativeStickyAssignor, RangeAssignor]` (cooperative *and* an old strategy).
2. Wait for entire group to converge.
3. Roll again with only `CooperativeStickyAssignor`.

> [!NOTE]
> Kafka Streams uses its own assignor (`StreamsPartitionAssignor`) which has been cooperative for longer than the plain consumer. Same eventual destination, slightly different lineage.

### Interview Follow-ups

- *"What's the difference between sticky and cooperative?"* — Sticky is about *minimizing partition movement*; cooperative is about *not stopping the world*. The default `CooperativeStickyAssignor` does both.
- *"What is static membership?"* — `group.instance.id` makes a consumer's identity survive restarts, avoiding a rebalance for transient outages (within `session.timeout.ms`).
- *"How does KIP-848 affect client compatibility?"* — Old clients keep using the classic protocol against the same coordinator; brokers support both.
