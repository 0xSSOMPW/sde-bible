# Q: Explain Brokers, Topics, and Partitions in Kafka.

**Answer:**

These are the three fundamental building blocks of Kafka's architecture.

### Broker
A **broker** is a single Kafka server. A Kafka cluster consists of multiple brokers (typically 3+). Each broker:
- Stores data on disk
- Serves producer and consumer requests
- Participates in replication
- Is identified by a unique integer ID

Brokers are designed so that **no single broker holds all the data** for a topic — data is distributed across brokers via partitions.

### Topic
A **topic** is a named category/feed to which messages are published. Think of it as a table in a database or a folder in a filesystem.

```
Topics: "user-signups", "order-events", "payment-transactions"
```

Topics are **multi-subscriber** — many consumer groups can read from the same topic independently without affecting each other.

### Partition
Each topic is split into one or more **partitions**. A partition is an **ordered, immutable sequence of messages** (an append-only log). Each message within a partition gets a sequential ID called an **offset**.

```
Topic: "orders" (3 partitions)

Partition 0: [msg0] [msg1] [msg2] [msg3] [msg4] → 
Partition 1: [msg0] [msg1] [msg2] →
Partition 2: [msg0] [msg1] [msg2] [msg3] →
```

### Why Partitions Matter

**1. Parallelism**
Each partition can be consumed by a different consumer in a consumer group. More partitions = more consumers = higher throughput.

**2. Ordering**
Messages are **strictly ordered WITHIN a partition**, but there is **no ordering guarantee ACROSS partitions**. If ordering matters for a specific entity (e.g., all events for user X), you must ensure all events for that entity go to the same partition using a partition key.

**3. Distribution**
Partitions are spread across brokers. For a topic with 6 partitions on a 3-broker cluster, each broker holds ~2 partitions.

### How They Relate

```
Kafka Cluster
├── Broker 0
│   ├── orders-partition-0 (Leader)
│   └── orders-partition-2 (Follower)
├── Broker 1
│   ├── orders-partition-1 (Leader)
│   └── orders-partition-0 (Follower)
└── Broker 2
    ├── orders-partition-2 (Leader)
    └── orders-partition-1 (Follower)
```

> [!IMPORTANT]
> Choosing the right number of partitions is a critical design decision. Too few = throughput bottleneck. Too many = increased memory usage, slower leader elections, and longer recovery times. A common starting point is **number of partitions = desired throughput / throughput per partition** (usually a few MB/s per partition).
