# Q: What is Schema Registry and why is Avro commonly used with Kafka?

**Answer:**

### The Problem: Schema Evolution
In a microservices architecture, producers and consumers are developed by different teams and deployed at different times. What happens when the producer changes the message format (adds a field, renames one, changes a type)?

Without schema management, the consumer **breaks** because it can't deserialize the new format.

### Schema Registry
The **Confluent Schema Registry** is a centralized service that stores and manages schemas for Kafka message keys and values. It ensures that producers and consumers agree on the data format.

```
Producer → Schema Registry: "Here's my schema, give me an ID"
           Schema Registry → "Schema ID: 42"
Producer → Kafka: [Schema ID: 42] + [Serialized Data]
                       ...
Consumer ← Kafka: [Schema ID: 42] + [Serialized Data]
Consumer → Schema Registry: "What schema is ID 42?"
           Schema Registry → Returns the schema
Consumer: Deserializes data using the schema
```

### Why Avro?
**Apache Avro** is a binary serialization format that is the dominant choice for Kafka messages. It pairs perfectly with Schema Registry.

**Avro schema example:**
```json
{
    "type": "record",
    "name": "OrderEvent",
    "namespace": "com.example",
    "fields": [
        {"name": "orderId", "type": "string"},
        {"name": "amount", "type": "double"},
        {"name": "currency", "type": "string", "default": "USD"},
        {"name": "timestamp", "type": "long"}
    ]
}
```

### Why not JSON?
| Feature | JSON | Avro | Protobuf |
|---|---|---|---|
| **Size** | Large (text + keys) | Compact (binary, no keys) | Compact (binary) |
| **Schema** | None (schema-less) | Required | Required |
| **Speed** | Slow (parsing text) | Fast (binary) | Fast (binary) |
| **Schema evolution** | Manual | Built-in | Built-in |
| **Human readable** | ✅ Yes | ❌ No | ❌ No |

Avro messages are typically **50-70% smaller** than JSON because they don't include field names — only the values, referenced by schema position.

### Compatibility Modes
Schema Registry enforces **compatibility rules** when a schema evolves:

| Mode | Rule |
|---|---|
| **BACKWARD** (default) | New schema can read old data. Allows: adding fields with defaults, removing fields. |
| **FORWARD** | Old schema can read new data. Allows: removing fields, adding optional fields. |
| **FULL** | Both backward and forward compatible. |
| **NONE** | No compatibility checks. |

**Example of backward-compatible change:**
```json
// v1
{"name": "orderId", "type": "string"}
{"name": "amount", "type": "double"}

// v2 (backward compatible: new field has a default)
{"name": "orderId", "type": "string"}
{"name": "amount", "type": "double"}
{"name": "currency", "type": "string", "default": "USD"}  // ← NEW
```

> [!TIP]
> In interviews, mentioning **Avro + Schema Registry** together shows you understand production Kafka. The key insight: Schema Registry acts as a contract between services, preventing breaking changes from deploying to production.
