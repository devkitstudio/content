## The Logical Replication Disk Exhaustion

While Change Data Capture (CDC) provides real-time syncing, a disconnected consumer can cascade into a catastrophic database outage.

### The WAL Hoarding Scenario

Debezium uses a PostgreSQL `Logical Replication Slot` to track consumed Write-Ahead Log (WAL) segments. If the Debezium connector crashes, the Kafka cluster goes down, or the network partitions, Debezium stops acknowledging offsets.

**The architectural flaw:** By default, PostgreSQL prioritizes replication integrity over its own survival. It will indefinitely hoard unacknowledged WAL files on the primary disk. If the outage persists, the disk reaches 100% capacity, and the entire PostgreSQL instance crashes.

### The Fix: Fail-Safe WAL Limits (PostgreSQL 13+)

Never deploy logical replication without a hard circuit breaker for WAL retention. You must instruct PostgreSQL to drop the slot to save itself if a consumer is dead.

```ini
# postgresql.conf
# DO NOT hardcode arbitrary sizes.
# Calculate: Average WAL Generation Rate per Hour * Maximum Acceptable Outage Hours
max_slot_wal_keep_size = <Calculated_Fail_Safe_Threshold>
```

**The Trade-off:** If this limit is breached, PostgreSQL survives, but the replication slot is permanently dropped. The CDC pipeline is broken. Upon recovery, Debezium will fail and must be re-provisioned to perform a highly expensive, full historical snapshot (`snapshot.mode = initial`).

### The Observability Mandate

Do not wait for standard Disk Space alerts. You must monitor the replication slot directly. Trigger a critical alert if:

1. The `active` state in `pg_replication_slots` becomes `false`.
2. The replication lag (the delta between `restart_lsn` and `pg_current_wal_lsn()`) exceeds your expected baseline threshold.
