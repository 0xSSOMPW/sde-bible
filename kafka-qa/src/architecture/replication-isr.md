# Q: How does Replication work in Kafka? What is the ISR?

**Answer:**

Replication is how Kafka achieves **fault tolerance**. Each partition is replicated across multiple brokers.

### Key Concepts

**Replication Factor**: The number of copies of each partition. A replication factor of 3 means every partition has 3 replicas across 3 different brokers.

**Leader Replica**: One replica is designated the **leader**. All producer writes and consumer reads go through the leader.

**Follower Replicas**: The remaining replicas continuously fetch new messages from the leader to stay in sync. They don't serve client requests (by default).

```
Topic: "payments" (replication-factor=3)

Broker 0: [Partition 0 - LEADER]    [Partition 1 - Follower]
Broker 1: [Partition 0 - Follower]  [Partition 1 - LEADER]
Broker 2: [Partition 0 - Follower]  [Partition 1 - Follower]
```

### What is the ISR (In-Sync Replicas)?

The **ISR** is the set of replicas that are "caught up" with the leader — they have replicated all messages within the allowed lag threshold (`replica.lag.time.max.ms`, default 30s).

```
Partition 0:
  Leader (Broker 0):   offset 100
  Follower (Broker 1): offset 99   ← In ISR (close enough)
  Follower (Broker 2): offset 85   ← NOT in ISR (too far behind)

ISR = {Broker 0, Broker 1}
```

### Why ISR Matters

The ISR directly affects **data durability** and **availability**:

1. **With `acks=all`**: The producer considers a write successful only when ALL replicas in the ISR have acknowledged it. If the ISR shrinks to just the leader, `acks=all` effectively becomes `acks=1`.

2. **Leader Election**: When a leader fails, the new leader is chosen **from the ISR** (by default). This ensures no data loss because ISR members have all committed messages.

3. **`min.insync.replicas`**: A critical safety net. If set to 2 (with replication-factor=3), the producer will refuse to write if the ISR drops below 2 replicas. This prevents data loss scenarios.

### Common Production Configuration

```properties
# Topic-level
replication.factor=3
min.insync.replicas=2

# Producer-level
acks=all
```

This means:
- 3 copies of every partition.
- At least 2 must acknowledge before a write is confirmed.
- If 2 brokers die, writes are rejected (protecting data integrity over availability).

> [!CAUTION]
> Setting `unclean.leader.election.enable=true` allows an out-of-sync replica to become leader when all ISR members are dead. This **guarantees availability** but **risks data loss** because the new leader may be missing messages. In most production systems, this is set to `false`.
