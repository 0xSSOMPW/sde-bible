# Q: How do you evolve schemas safely with Schema Registry?

**Answer:**

Schema Registry stores Avro/Protobuf/JSON schemas keyed by subject (usually `<topic>-value`). Compatibility rules determine which schema changes are allowed. Picking the wrong compatibility mode is how you ship a deserialization-error incident.

### The Five Compatibility Modes

| Mode | New schema can | Use when |
|------|---------------|----------|
| `BACKWARD` (default) | Read old data | Consumers upgrade first |
| `FORWARD` | Old schema can read new data | Producers upgrade first |
| `FULL` | Both: BACKWARD + FORWARD | You want symmetric safety |
| `*_TRANSITIVE` | Same, but compared against **all** historical versions | Long-lived topics |
| `NONE` | Anything | Test only — don't ship |

The default `BACKWARD` is the right choice for most consumer-led rollouts. `BACKWARD_TRANSITIVE` is safer but stricter.

### What Each Mode Actually Allows

**BACKWARD** (consumer with new schema reads data written with old schema):
- ✅ Add an optional field (with default).
- ✅ Remove an optional field.
- ❌ Add a required field (consumer can't read old data missing it).
- ❌ Rename a field (different name = different field).
- ❌ Change a field's type incompatibly.

**FORWARD** (consumer with old schema reads data written with new schema):
- ✅ Add a required field (old consumer just ignores it).
- ✅ Remove an optional field with default.
- ❌ Remove a required field (old consumer fails to find it).

**FULL** = intersection of both. The strictest practical rule.

### Worked Examples

Initial schema:

```json
{
  "type": "record", "name": "Order", "fields": [
    {"name": "id",     "type": "string"},
    {"name": "amount", "type": "double"}
  ]
}
```

#### BACKWARD: add an optional field

```json
{
  "type": "record", "name": "Order", "fields": [
    {"name": "id",     "type": "string"},
    {"name": "amount", "type": "double"},
    {"name": "tax",    "type": ["null", "double"], "default": null}
  ]
}
```

✅ Old records (no `tax`) → new consumer reads `tax=null`.
✅ Schema Registry accepts.

#### Failure: required field with no default

```json
{"name": "tax", "type": "double"}    // no default
```

❌ Backward-incompatible. Old data has no `tax` field; new consumer can't supply one.

Schema Registry rejects with `409 Conflict` at registration time.

### Subject Naming Strategies

- **TopicNameStrategy** (default): `<topic>-key`, `<topic>-value`. One schema per topic.
- **RecordNameStrategy**: subject = fully-qualified record name. Lets one topic carry multiple record types.
- **TopicRecordNameStrategy**: `<topic>-<recordName>`. Hybrid.

Use `RecordNameStrategy` when you want event-typed topics (e.g., `OrderCreated`, `OrderShipped` on the same `orders` topic).

### The Magic Byte Wire Format

Avro/Proto-over-Kafka uses Confluent's wire format:

```
| 0x00 | schema_id (4 bytes, big-endian) | payload bytes |
   ^         ^                                ^
   magic     schema fetched from Registry     Avro/Proto bytes
```

Consumer sees the schema ID, fetches the writer's schema from Registry (cached), then deserializes against its own reader's schema. No schema travels with each message — that's why Registry is mandatory.

### Schema Registry Operations

```bash
# Register a schema
curl -X POST -H "Content-Type: application/json" \
  --data '{"schema": "{\"type\": \"record\", ...}"}' \
  http://registry:8081/subjects/orders-value/versions

# Get latest schema
curl http://registry:8081/subjects/orders-value/versions/latest

# Set compatibility mode for a subject
curl -X PUT -H "Content-Type: application/json" \
  --data '{"compatibility": "BACKWARD_TRANSITIVE"}' \
  http://registry:8081/config/orders-value
```

### CI Gates

The single most valuable thing you can do: register schemas as a CI step **before merge**.

```bash
# Maven plugin
mvn schema-registry:test-compatibility \
  -Dschema.registry.url=https://registry \
  -DsubjectNamingStrategy=TopicNameStrategy
```

If the proposed schema is incompatible, the build fails. Production never sees the bad schema.

### Removing or Renaming a Field

You don't. You **deprecate**:

1. Make the field optional with a default (BACKWARD-compatible change).
2. Update consumers to stop relying on it.
3. Update producers to stop writing it.
4. Leave the field in the schema indefinitely (or remove on a coordinated "version 2" topic migration).

To rename, add the new name as an optional field, dual-write, switch readers, retire old.

Avro **aliases** let you rename without breaking readers:

```json
{"name": "amount", "type": "double", "aliases": ["price"]}
```

Reader looking for `price` finds `amount`. Limited usefulness; not all serializers honor aliases.

### Reference Schemas

Avoid duplicating common types. Register a `Money` schema, reference it:

```json
{
  "type": "record", "name": "Order",
  "fields": [
    {"name": "total", "type": "com.example.Money"}
  ]
}
```

Registered with `references = [{"name": "com.example.Money", "subject": "money-value", "version": 1}]`.

### Protobuf vs Avro vs JSON Schema

| Aspect | Avro | Protobuf | JSON Schema |
|--------|------|----------|-------------|
| Wire size | Smallest (no field names) | Small (tag numbers) | Largest (field names) |
| Schema travels with data | No (Registry) | No (Registry) | Optional |
| Tooling | Strong on JVM, OK elsewhere | Excellent everywhere | Universal |
| Schema evolution | Field-by-name, with defaults | Field-by-tag-number | Field-by-name |
| Required vs optional | Default-aware | All optional in proto3 | `required` array |
| Use case | Analytics-heavy, JVM-heavy | Polyglot service-to-service | Human-readable, web-friendly |

Avro is the historical Kafka default; Protobuf has caught up; JSON Schema is convenient but bulky.

### Operational Concerns

- **Registry HA**: run ≥ 2 instances. Schema IDs are global; ID allocation is leader-only.
- **Cache schemas client-side**: `client.cache.capacity` (default 1000). A single schema fetch per ID per consumer lifetime is cheap.
- **Schema deletion**: soft-delete removes from API but keeps the ID alive (consumers can still deserialize). Hard-delete loses the ability to deserialize old data — almost never the right call.

### Common Mistakes

| Mistake | Fix |
|---------|-----|
| Required field added → backward-incompatible | Always add fields with default values |
| Compatibility mode `NONE` "for now" | One step from a prod incident |
| One subject per microservice, mixed event types | Use `RecordNameStrategy` if multiple types share a topic |
| Treating schema changes like code | Schema changes are *contract* changes — require migration coordination |
| Forgetting `TRANSITIVE` on long-lived topics | Without it, only the previous version is checked |

> [!NOTE]
> Avro/Proto defaults rule: every new field should have a default. Treat removal as "make optional, leave forever." Treat rename as "add new, deprecate old."

### Interview Follow-ups

- *"What happens if Schema Registry is down?"* — Producers can use cached schemas; new schemas can't register. Consumers can deserialize cached IDs but new ones fail. Multi-AZ HA is essential.
- *"How do you delete a topic that uses a schema?"* — Delete topic in Kafka, optionally delete subject in Registry. ID assignments are still tracked.
- *"Can two producers race-register the same schema?"* — Registry deduplicates by content hash — same schema = same ID, no race.
