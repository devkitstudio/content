## The Deadly Trap of Replication Slots

While Debezium provides real-time syncing, a misconfigured PostgreSQL replication slot can take down your entire production database.

### The Disk Exhaustion Scenario

Debezium creates a `Logical Replication Slot` in PostgreSQL to keep track of which WAL (Write-Ahead Log) segments it has consumed.

If the Debezium container crashes, or the Kafka cluster goes down, Debezium stops acknowledging messages. **PostgreSQL will refuse to delete old WAL files**, hoarding them on disk until the Debezium consumer returns. If the outage lasts too long, the disk reaches 100% capacity, and PostgreSQL crashes completely.

### The Fix: Enforce WAL Limits (PostgreSQL 13+)

Never deploy CDC without setting a hard limit on WAL retention. If Debezium is offline for too long, PostgreSQL will automatically drop the slot to save itself.

```ini
# In postgresql.conf
# Limit the WAL size the replication slot can retain (e.g., 50GB)
max_slot_wal_keep_size = 50GB
```

_Note: If the slot is dropped due to this limit, Debezium will crash upon reconnecting and must be reconfigured to take a fresh full snapshot (`snapshot.mode = initial`)._

### Monitoring Requirement

Always monitor the `pg_replication_slots` view. Trigger a PagerDuty alert if the `active` state becomes `false` or if `restart_lsn` lags significantly.
