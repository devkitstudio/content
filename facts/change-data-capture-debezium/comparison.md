## Data Synchronization Patterns

Replicating state from a primary database to a secondary data store (like Elasticsearch or a Data Warehouse) requires architectural trade-offs.

| Pattern               | How It Works                                                  | Trade-offs                                                                                                                                  | Best For                                                                    |
| :-------------------- | :------------------------------------------------------------ | :------------------------------------------------------------------------------------------------------------------------------------------ | :-------------------------------------------------------------------------- |
| **Dual Writes**       | App writes to DB, then sends event to Message Queue.          | ❌ High risk of data inconsistency (split-brain) if the second write fails. Hard to maintain.                                               | Avoid in production unless using distributed transactions (Saga).           |
| **Timestamp Polling** | App runs `SELECT * FROM table WHERE updated_at > last_check`. | ❌ Misses hard `DELETE` operations. Places heavy CPU/I/O load on the database.                                                              | Legacy systems with no CDC support. Low-frequency batch syncs.              |
| **CDC (Debezium)**    | Reads database transaction logs asynchronously.               | ✅ Zero impact on DB query performance. Captures deletes perfectly. <br>❌ Requires heavy infrastructure (Kafka, Zookeeper/KRaft, Connect). | High-throughput systems requiring strict consistency and real-time syncing. |
