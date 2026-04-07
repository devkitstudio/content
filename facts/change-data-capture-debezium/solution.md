## CDC Architecture

```
PostgreSQL (WAL logs)
    ↓
Debezium PostgreSQL Connector
    ↓
Kafka Topics (change events)
    ↓
Elasticsearch Sink Connector
    ↓
Elasticsearch (synced index)
```

### How It Works

Debezium reads PostgreSQL Write-Ahead Logs (WAL) to capture every change:

```
INSERT INTO users VALUES (1, 'Alice')
    ↓ (written to WAL)
    ↓
Debezium reads WAL
    ↓
Produces to Kafka: { "op": "c", "after": { "id": 1, "name": "Alice" } }
    ↓
Elasticsearch Sink reads Kafka
    ↓
Elasticsearch index updated
```

### Configuration

```yaml
# debezium-postgres-connector.json
{
  "name": "postgres-cdc",
  "config": {
    "connector.class": "io.debezium.connector.postgresql.PostgresConnector",
    "database.server.name": "postgres_prod",
    "database.hostname": "postgres.example.com",
    "database.port": 5432,
    "database.user": "replicator",
    "database.password": "secret",
    "database.dbname": "app_db",
    "plugin.name": "pgoutput",  # or decoderbufs
    "publication.name": "debezium_publication",
    "slot.name": "debezium_slot",
    "table.include.list": "public.users,public.orders",
    "topic.prefix": "cdc",
    "key.converter": "org.apache.kafka.connect.json.JsonConverter",
    "value.converter": "org.apache.kafka.connect.json.JsonConverter",
    "snapshot.mode": "initial",
    "schema.history.internal": "io.debezium.relational.history.FileSchemasHistory",
    "schema.history.internal.file.filename": "/schema-history"
  }
}
```

### Setup PostgreSQL

```sql
-- Enable logical replication
ALTER SYSTEM SET wal_level = logical;

-- Restart PostgreSQL (requires restart of database service)
-- sudo systemctl restart postgresql

-- Create replication user
CREATE ROLE replicator WITH LOGIN PASSWORD 'secret' REPLICATION;

-- Create publication (Postgres 10+)
CREATE PUBLICATION debezium_publication FOR ALL TABLES;

-- Verify
SELECT * FROM pg_stat_replication;
```

### Kafka Message Format

```json
{
  "schema": {
    "type": "struct",
    "fields": [
      { "name": "id", "type": "int64" },
      { "name": "name", "type": "string" },
      { "name": "email", "type": "string" }
    ]
  },
  "payload": {
    "before": null,
    "after": {
      "id": 1,
      "name": "Alice",
      "email": "alice@example.com"
    },
    "source": {
      "version": "2.1.0",
      "connector": "postgresql",
      "name": "postgres_prod",
      "ts_ms": 1234567890,
      "txId": 123,
      "lsn": 456789,
      "xmin": null
    },
    "op": "c",  # "c"=create, "u"=update, "d"=delete, "r"=read
    "ts_ms": 1234567891,
    "transaction": null
  }
}
```

### Elasticsearch Sink

```yaml
# debezium-elasticsearch-sink.json
{
  "name": "elasticsearch-sink",
  "config": {
    "connector.class": "io.confluent.connect.elasticsearch.ElasticsearchSinkConnector",
    "connection.url": "http://elasticsearch:9200",
    "connection.username": "elastic",
    "connection.password": "changeme",
    "type.name": "_doc",
    "topics": "cdc.public.users,cdc.public.orders",
    "key.ignore": false,
    "schema.ignore": false,
    "write.method": "upsert",
    "transforms": "unwrap",
    "transforms.unwrap.type": "io.debezium.transforms.ExtractNewRecordState",
    "transforms.unwrap.drop.tombstones": true
  }
}
```

### Processing CDC Events in Application

```python
from kafka import KafkaConsumer
import json
from elasticsearch import Elasticsearch

class CDCProcessor:
    def __init__(self):
        self.consumer = KafkaConsumer(
            'cdc.public.users',
            bootstrap_servers=['localhost:9092'],
            value_deserializer=lambda m: json.loads(m.decode('utf-8'))
        )
        self.es = Elasticsearch(['localhost:9200'])

    def process_events(self):
        for message in self.consumer:
            event = message.value
            payload = event['payload']

            user_id = payload['after']['id']
            operation = payload['op']

            if operation in ('c', 'u'):  # create or update
                self.es.index(
                    index='users',
                    id=user_id,
                    body=payload['after']
                )
            elif operation == 'd':  # delete
                self.es.delete(index='users', id=user_id)

            print(f"Synced {operation} operation for user {user_id}")

processor = CDCProcessor()
processor.process_events()
```

## Advantages

| Feature | Manual Sync | CDC + Debezium |
|---------|------------|----------------|
| Latency | Minutes | <1 second |
| Missed updates | Common | Zero |
| CPU impact | Medium (polling) | Low (log-based) |
| Schema evolution | Manual | Automatic |
| Ordering | Not guaranteed | Guaranteed per table |
