# CDC Architecture with Debezium & Kafka

Traditional database polling (`SELECT * WHERE updated_at > ?`) introduces heavy load and misses hard deletions. Change Data Capture (CDC) eliminates this by reading directly from the database's internal transaction log.

## 1. The Pipeline Architecture

```text
[PostgreSQL] ──(WAL Logs)──▶ [Debezium Source] ──▶ [Kafka Topic] ──▶ [Elasticsearch Sink] ──▶ [Search Index]
```

Debezium tails the PostgreSQL Write-Ahead Log (WAL) via logical replication. Every `INSERT`, `UPDATE`, or `DELETE` is captured exactly once and published to Kafka as an immutable event stream.

## 2. Core PostgreSQL Configuration

To allow Debezium to read the replication stream, PostgreSQL requires specific settings in `postgresql.conf`:

```ini
# Required for Debezium CDC
wal_level = logical
max_wal_senders = 1  # Must be >= 1
max_replication_slots = 1 # Must be >= 1
```

## 3. Kafka Connect Payload (Debezium JSON)

Deploying the connector via Kafka Connect REST API. Notice the `ExtractNewRecordState` transform, which unwraps the complex Debezium payload into a clean JSON object for downstream systems.

```json
{
  "name": "postgres-cdc",
  "config": {
    "connector.class": "io.debezium.connector.postgresql.PostgresConnector",
    "database.hostname": "postgres.internal",
    "database.port": "5432",
    "database.user": "replicator",
    "database.password": "secret",
    "database.dbname": "app_db",
    "database.server.name": "pg_prod",
    "plugin.name": "pgoutput",
    "table.include.list": "public.users",
    "topic.prefix": "cdc",
    "snapshot.mode": "initial",

    // Unwraps the nested payload and automatically handles DELETE tombstones
    "transforms": "unwrap",
    "transforms.unwrap.type": "io.debezium.transforms.ExtractNewRecordState",
    "transforms.unwrap.drop.tombstones": false,
    "transforms.unwrap.delete.handling.mode": "rewrite"
  }
}
```
