## Autovacuum Tuning by Workload

### Small Database (< 10GB, low UPDATE rate)
```ini
autovacuum_naptime = 30s
autovacuum_vacuum_threshold = 100
autovacuum_vacuum_scale_factor = 0.1
autovacuum_max_workers = 1
```

### Medium Database (10GB-100GB, moderate UPDATE)
```ini
autovacuum_naptime = 10s
autovacuum_vacuum_threshold = 1000
autovacuum_vacuum_scale_factor = 0.05
autovacuum_analyze_threshold = 500
autovacuum_analyze_scale_factor = 0.02
autovacuum_max_workers = 2
```

### Large Database (100GB+, heavy UPDATE)
```ini
autovacuum_naptime = 5s          # Check very frequently
autovacuum_vacuum_threshold = 5000
autovacuum_vacuum_scale_factor = 0.01   # 1% of table
autovacuum_vacuum_cost_delay = 5ms      # Don't throttle
autovacuum_vacuum_cost_limit = 20000
autovacuum_max_workers = 4
autovacuum_work_mem = 512MB
```

### Very High Traffic (millions of updates/sec)
```ini
autovacuum_naptime = 1s
autovacuum_vacuum_threshold = 50000
autovacuum_vacuum_scale_factor = 0.005
autovacuum_vacuum_cost_delay = 0  # Don't throttle at all
autovacuum_vacuum_cost_limit = 50000
autovacuum_max_workers = 8
autovacuum_work_mem = 1GB

-- Per-table for hottest tables
ALTER TABLE events SET (
    autovacuum_vacuum_threshold = 100000,
    autovacuum_vacuum_scale_factor = 0.001
);
```

## Parameter Explanations

| Parameter | Default | Meaning | Adjust |
|-----------|---------|---------|--------|
| `autovacuum_naptime` | 10s | How often to check | Decrease = more frequent checks |
| `autovacuum_vacuum_threshold` | 50 | Minimum dead tuples | Increase on huge tables (100GB+) |
| `autovacuum_vacuum_scale_factor` | 0.1 | % of live tuples triggering vacuum | Decrease = more aggressive (0.01 for busy) |
| `autovacuum_analyze_threshold` | 50 | Minimum dead tuples for analyze | Usually ~50% of vacuum threshold |
| `autovacuum_max_workers` | 3 | Parallel vacuum processes | Increase on multi-core servers |
| `autovacuum_work_mem` | -1 (use maintenance_work_mem) | Memory per worker | Increase on large RAM servers |
| `autovacuum_vacuum_cost_delay` | 20ms | Delay between page scans | Decrease to less throttle (0 = none) |
| `autovacuum_vacuum_cost_limit` | 200 | Pages scanned per cost_delay | Increase for faster vacuum |

## Diagnostic Queries

```sql
-- Check autovacuum activity
SELECT datname, schemaname, relname,
       last_vacuum, last_autovacuum,
       vacuum_count, autovacuum_count
FROM pg_stat_all_tables
WHERE last_autovacuum > now() - interval '1 hour'
ORDER BY last_autovacuum DESC;

-- Find tables with excessive dead tuples (last_vacuum was long ago)
SELECT schemaname, tablename, n_dead_tup,
       EXTRACT(HOURS FROM (now() - last_vacuum)) as hours_since_vacuum,
       EXTRACT(HOURS FROM (now() - last_autovacuum)) as hours_since_autovacuum
FROM pg_stat_user_tables
WHERE n_dead_tup > 100000
ORDER BY n_dead_tup DESC;

-- Check autovacuum worker status
SELECT pid, query, query_start, state
FROM pg_stat_activity
WHERE query LIKE '%autovacuum%'
ORDER BY query_start;

-- Index bloat from dead tuples
SELECT schemaname, indexname, idx_blks_read, idx_blks_hit, idx_scan, idx_tup_read, idx_tup_fetch
FROM pg_stat_user_indexes
WHERE idx_blks_read > 1000  -- Lots of disk reads = bloated
ORDER BY idx_blks_read DESC;
```

## Preventing Bloat with Monitoring

```bash
#!/bin/bash
# monitor_vacuum.sh - Run daily via cron

psql -h localhost -U postgres -d mydb <<EOF
-- Alert if tables have >20% dead tuples
SELECT schemaname, tablename, n_dead_tup, n_live_tup,
       ROUND(100.0 * n_dead_tup / (n_live_tup + n_dead_tup), 1) as dead_pct
FROM pg_stat_user_tables
WHERE n_live_tup > 100000
  AND n_dead_tup > (n_live_tup * 0.2)
ORDER BY dead_pct DESC;
EOF

# If query returns results, table needs attention
if [ $? -eq 0 ]; then
    echo "Alert: Tables have excessive dead tuples" | mail -s "DB Maintenance Alert" ops@example.com
fi
```

## VACUUM FULL Recovery Plan

```sql
-- If database is 100% full and VACUUM FULL is too slow:

-- 1. Move data to temporary table (in batches)
CREATE TABLE users_new AS
SELECT * FROM users LIMIT 1000000;

-- 2. Drop original table
DROP TABLE users CASCADE;

-- 3. Rename temporary
ALTER TABLE users_new RENAME TO users;

-- 4. Recreate indexes
CREATE INDEX idx_users_email ON users(email);

-- 5. Restore permissions
GRANT SELECT, INSERT, UPDATE, DELETE ON users TO app_user;

-- Full downtime: ~1 hour for 100GB
-- Used only when VACUUM FULL takes too long (disk critically full)
```

## Autovacuum vs Manual VACUUM

| Aspect | Autovacuum | Manual VACUUM |
|--------|-----------|---------------|
| When | Triggered by threshold | On schedule |
| Locking | Concurrent (minimal) | Non-blocking (default) |
| Cost | Throttled (configurable) | Full speed |
| Best for | Ongoing maintenance | Heavy workloads |
| Risk | Can interfere with queries | Can fill disk if not run |
| Recommendation | Always ON | Run nightly as safety |

**Best practice: Both**
```sql
-- Autovacuum for continuous cleanup
-- Manual VACUUM ANALYZE nightly during maintenance window

-- In cron (every night at 2 AM):
0 2 * * * psql -U postgres -d mydb -c "VACUUM ANALYZE;"
```
