# Q: What is the difference between Retention and Log Compaction in Kafka?

**Answer:**

Kafka keeps data on disk and provides two distinct retention strategies for controlling how long messages are stored.

### Time/Size-Based Retention (Default)
Messages are deleted after a configured time period or when the log exceeds a size limit.

```properties
# Time-based: Delete messages older than 7 days
log.retention.hours=168        # (default: 168 hours = 7 days)

# Size-based: Delete oldest messages when partition log exceeds 1GB
log.retention.bytes=1073741824

# Segment file size (retention is applied per segment)
log.segment.bytes=1073741824   # 1GB per segment file
```

**How it works:** Kafka stores messages in segment files. When a segment's age exceeds `retention.hours` (or the total log size exceeds `retention.bytes`), the entire segment file is deleted.

```
Partition 0:
  segment-0.log (2 days old) ← DELETED when > 7 days
  segment-1.log (1 day old)
  segment-2.log (current, active)
```

### Log Compaction
Instead of deleting messages by time, Kafka keeps **only the latest value for each unique key**. It's as if you have a table where each key's row is updated in-place.

```properties
log.cleanup.policy=compact
```

**Before compaction:**
```
Offset  Key      Value
0       user-1   {"name": "Alice"}
1       user-2   {"name": "Bob"}
2       user-1   {"name": "Alice Smith"}  ← newer value for user-1
3       user-3   {"name": "Charlie"}
4       user-2   null                     ← tombstone (delete marker)
```

**After compaction:**
```
Offset  Key      Value
2       user-1   {"name": "Alice Smith"}  ← latest value kept
3       user-3   {"name": "Charlie"}
                                          ← user-2 deleted (tombstone)
```

### When to Use Which?

| Strategy | Policy | Use Case |
|---|---|---|
| **Time/Size Retention** | `delete` (default) | Event streams, logs, metrics (you care about events over time) |
| **Log Compaction** | `compact` | State snapshots, CDC changes, config updates (you care about latest state per key) |
| **Both** | `compact,delete` | Compacted but also enforce a time limit on old keys |

### Real-World Examples

**Compaction:**
- `__consumer_offsets` — Kafka's internal topic for consumer offsets (only latest offset per group/partition matters).
- CDC topics (Debezium) — latest row state per primary key.
- User profile cache — latest profile per user ID.

**Time retention:**
- Clickstream events, application logs, order events.

> [!IMPORTANT]
> Compaction is **not immediate**. A background thread called the "log cleaner" periodically compacts segments. Between compactions, both old and new values for a key may exist. Never rely on compaction for real-time deduplication — it's an eventual cleanup mechanism.
