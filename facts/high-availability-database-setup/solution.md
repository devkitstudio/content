## HA Objectives

- **RTO** (Recovery Time Objective): Failover < 10 seconds
- **RPO** (Recovery Point Objective): Lose < 1 second of data

**Without HA:**
- Primary down → no writes → app broken
- Failover manual → 5+ minutes
- Data loss → conflicts in recovery

## Architecture: Primary + Streaming Replicas + Patroni

```
┌──────────────────────────────────────────────┐
│  Application                                  │
│  (uses VIP: db.internal)                     │
└──────────┬───────────────────────────────────┘
           │
    ┌──────▼──────────────────────────┐
    │  Patroni (HA Manager)            │
    │  Watches primary, auto-failover  │
    └──────┬──────────────────────────┘
           │
    ┌──────┴─────────┬────────────────┐
    │                │                │
┌───▼────────┐  ┌───▼───────┐  ┌───▼───────┐
│ Primary    │  │ Replica 1 │  │ Replica 2 │
│ (WRITES)   │  │ (READ)    │  │ (STANDBY) │
└────────────┘  └───────────┘  └───────────┘
  streaming      streaming
  replication    replication
```

## Streaming Replication

Primary continuously sends WAL (Write-Ahead Log) to replicas.

```sql
-- On primary: Configure replication
ALTER SYSTEM SET max_wal_senders = 5;
ALTER SYSTEM SET wal_keep_segments = 64;
ALTER SYSTEM SET hot_standby = on;
SELECT pg_reload_conf();

-- On primary: Create replication user
CREATE ROLE replicator WITH LOGIN REPLICATION PASSWORD 'secret123';

-- On replica: Create standby from backup
pg_basebackup -h primary.local -D /var/lib/postgresql/data -U replicator

-- On replica: Create recovery.conf
echo "standby_mode = 'on'
primary_conninfo = 'host=primary.local user=replicator password=secret123'
" > /var/lib/postgresql/recovery.conf

-- Start replica
systemctl start postgresql
```

## Synchronous vs Asynchronous Replication

**Asynchronous (default, RPO ~10-30 seconds):**
```
Primary writes → commits → replica catches up later
↓
Fast writes, potential data loss
```

**Synchronous (RPO ~0, but slower):**
```sql
ALTER SYSTEM SET synchronous_commit = 'remote_apply';
-- Primary waits for replica confirmation before commit
-- Slower but no data loss
```

**Hybrid (recommended):**
```sql
ALTER SYSTEM SET synchronous_commit = 'remote_write';
-- Replica writes to OS buffer (no fsync on replica)
-- Faster than remote_apply, good data safety
```

## Automatic Failover with Patroni

Patroni manages failover automatically.

```yaml
# /etc/patroni/patroni.yml
scope: my-postgres-cluster
namespace: /patroni/
name: pg-node-1

postgresql:
  data_dir: /var/lib/postgresql/data
  bin_dir: /usr/bin
  listen: 0.0.0.0:5432
  connect_address: 10.0.1.10:5432
  create_replica_methods:
    - pg_basebackup
  pg_basebackup:
    checkpoint: fast

dcs:
  type: etcd
  host: 10.0.1.5:2379
  ttl: 30
  loop_wait: 10
  retry_timeout: 10
  maximum_lag_on_failover: 1048576

restapi:
  listen: 0.0.0.0:8008

tags:
  nofailover: false
  noloadbalance: false
  clonefrom: false
```

**How it works:**
1. Patroni on each node registers in etcd
2. Patroni leader = primary, others = replicas
3. Primary heartbeat → etcd every 30s
4. No heartbeat → failover after 60s
5. Elect new primary from replicas
6. VIP moves to new primary

**VIP failover:**
```bash
# HAProxy or keepalived maintains VIP
# Application connects to db.internal (VIP)
# On failover, VIP automatically moves to new primary
# Applications reconnect automatically
```

## Failover Performance

```sql
-- Monitor replication lag
SELECT
  client_addr,
  pg_wal_lsn_diff(pg_current_wal_lsn(), flush_lsn) AS bytes_behind,
  ROUND(bytes_behind::numeric / 1024 / 1024, 2) AS MB_behind
FROM pg_stat_replication;
```

**Failover timeline:**
```
t=0s: Primary crashes
t=2-5s: Patroni detects no heartbeat
t=5-8s: Consensus reached, new primary elected
t=8-10s: VIP moves to new primary
t=10s: Applications reconnect, recovery complete

Total RTO: ~10 seconds
Data loss: < 1 second (lag)
```

## Monitoring & Alerts

```sql
-- Alert if replication lag > 1 second
CREATE TABLE replication_lag_alert (
  created_at TIMESTAMP DEFAULT NOW(),
  lag_bytes BIGINT,
  lag_seconds FLOAT
);

-- Query for monitoring
SELECT
  application_name,
  client_addr,
  state,
  sync_state,
  pg_wal_lsn_diff(pg_current_wal_lsn(), flush_lsn) as lag_bytes
FROM pg_stat_replication;

-- Failover readiness
SELECT pid, usename, application_name, state
FROM pg_stat_replication
WHERE sync_state = 'sync'
  AND state = 'streaming';
```

## Backup Strategy

```bash
# Continuous archiving + WAL backup
# Even with streaming replication, keep WAL archives

# Archive WAL every hour
archive_command = 'aws s3 cp %p s3://backups/postgres/wal/%f'

# Full backup daily to S3
pg_basebackup -h primary.local -D backup/ -X fetch
tar -czf postgres-$(date +%Y%m%d).tar.gz backup/
aws s3 cp postgres-$(date +%Y%m%d).tar.gz s3://backups/postgres/

# Point-in-time recovery capability
restore_command = 'aws s3 cp s3://backups/postgres/wal/%f %p'
```
