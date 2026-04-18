# Q: What are the different `acks` settings and how do they affect durability?

**Answer:**

The `acks` (acknowledgements) producer configuration controls **how many brokers must confirm receipt of a message** before the producer considers the write successful. It's the primary knob for trading off between **durability** and **latency**.

### `acks=0` (Fire and Forget)
The producer does not wait for any acknowledgement. It sends the message and immediately considers it delivered.

- **Durability**: None. Messages can be lost silently.
- **Latency**: Lowest possible.
- **Use case**: Metrics, logs, or any data where occasional loss is acceptable.

### `acks=1` (Leader Acknowledgement)
The producer waits for the **leader replica** to write the message to its local log and acknowledge. Followers may not have replicated it yet.

- **Durability**: Message is lost if the leader crashes before followers replicate.
- **Latency**: Low.
- **Use case**: General-purpose, acceptable for most non-critical workloads.

### `acks=all` (or `acks=-1`) (Full ISR Acknowledgement)
The producer waits for **all replicas in the ISR** to acknowledge. This is the strongest durability guarantee.

- **Durability**: Message survives as long as at least one ISR replica survives.
- **Latency**: Highest (waiting for multiple replicas).
- **Use case**: Financial transactions, order processing, anything where data loss is unacceptable.

### Visual Comparison

```
Producer → [Broker 0: Leader] → [Broker 1: Follower] → [Broker 2: Follower]

acks=0:  Producer sends, doesn't wait.        Risk: Total loss
acks=1:  Producer waits for Leader ACK.        Risk: Leader dies before replication
acks=all: Producer waits for ALL ISR ACKs.     Risk: Only if ALL replicas die
```

### The `min.insync.replicas` Safety Net
`acks=all` alone has a subtle trap: if the ISR shrinks to just the leader (all followers are lagging), then `acks=all` effectively becomes `acks=1`.

The fix is combining it with `min.insync.replicas`:
```properties
acks=all
min.insync.replicas=2  # At least 2 replicas must ACK
replication.factor=3
```

If fewer than 2 replicas are in the ISR, the producer receives a `NotEnoughReplicasException` and the write is rejected — preventing the silent durability downgrade.

> [!TIP]
> The gold standard production config is `acks=all` + `min.insync.replicas=2` + `replication.factor=3`. This tolerates one broker failure while guaranteeing no data loss.
