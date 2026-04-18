# Q: What is Kafka Connect?

**Answer:**

**Kafka Connect** is a framework for reliably streaming data **between Kafka and external systems** (databases, search indexes, filesystems, cloud services) without writing any code.

### How It Works
Kafka Connect runs as a separate, scalable cluster of **worker** processes. You configure data pipelines using JSON configurations — no Java code required.

```
                    Kafka Connect
External Source ──▶ [Source Connector] ──▶ Kafka Topic
Kafka Topic    ──▶ [Sink Connector]   ──▶ External Sink
```

### Source Connectors
Read data from an external system and write it to Kafka topics.

```json
{
    "name": "postgres-source",
    "config": {
        "connector.class": "io.debezium.connector.postgresql.PostgresConnector",
        "database.hostname": "db.example.com",
        "database.port": "5432",
        "database.dbname": "orders_db",
        "topic.prefix": "cdc"
    }
}
```
This captures every INSERT/UPDATE/DELETE from Postgres and streams it to topics like `cdc.public.orders`, `cdc.public.users`.

### Sink Connectors
Read data from Kafka topics and write it to an external system.

```json
{
    "name": "elasticsearch-sink",
    "config": {
        "connector.class": "io.confluent.connect.elasticsearch.ElasticsearchSinkConnector",
        "topics": "orders",
        "connection.url": "http://es.example.com:9200",
        "type.name": "_doc"
    }
}
```

### Popular Connectors

| Connector | Direction | Use Case |
|---|---|---|
| **Debezium (PostgreSQL/MySQL)** | Source | Change Data Capture (CDC) |
| **JDBC Connector** | Source/Sink | Generic SQL database sync |
| **Elasticsearch** | Sink | Search indexing |
| **S3 Sink** | Sink | Data lake / archival |
| **BigQuery Sink** | Sink | Analytics warehouse |
| **File Stream** | Source/Sink | CSV/log file ingestion |

### Standalone vs Distributed Mode

| Mode | Workers | Use Case |
|---|---|---|
| **Standalone** | 1 | Development, testing |
| **Distributed** | Multiple | Production (fault-tolerant, scalable) |

In distributed mode, if a worker dies, its connectors are automatically reassigned to surviving workers.

### Why Not Just Write a Custom Producer/Consumer?
- **Built-in offset tracking** — Connect tracks source positions automatically.
- **Fault tolerance** — automatic failover in distributed mode.
- **Schema evolution** — integrates with Schema Registry.
- **Configurable transforms** — Single Message Transforms (SMTs) for lightweight data manipulation.
- **No code to maintain** — just JSON config.

> [!TIP]
> In interviews, **Debezium + Kafka Connect** for CDC is a particularly strong topic. It's the industry standard for streaming database changes (e.g., syncing a PostgreSQL write-model to an Elasticsearch read-model in real-time).
