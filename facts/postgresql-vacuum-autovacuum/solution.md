## How MVCC Creates Dead Tuples

PostgreSQL uses MVCC (Multi-Version Concurrency Control):
- UPDATE creates new row version, marks old as "dead"
- DELETE marks row as dead
- Other transactions might still see old version
- Once all transactions finish, row is garbage

**Example:**
```sql
-- Transaction 1: UPDATE rows
UPDATE users SET status = 'active' WHERE id IN (1,2,3,4,5);
-- 5 old row versions are now "dead" (unmarked, but taking space)

-- Without VACUUM: Dead tuples stay on disk forever
-- Disk usage: 5MB → 10MB → 50MB (after months of updates)
```

## VACUUM: Garbage Collection

**VACUUM (default)**
```sql
-- Marks dead tuples as reusable space
-- Reads entire table, builds free space map
-- Takes minutes on large tables
-- Doesn't release space back to OS

VACUUM users;
```

**VACUUM FULL (aggressive)**
```sql
-- Rewrites entire table, shrinks file
-- Locks table for writes during process
-- Releases space back to OS immediately
-- 10-50x slower than regular VACUUM

VACUUM FULL users;  -- Takes 10+ minutes on 100GB table
```

**VACUUM ANALYZE (most common)**
```sql
-- VACUUM + update table statistics
-- Statistics used by query planner for optimization

VACUUM ANALYZE users;
```

## Why Disk Fills Up

```
Production server:
- 50M rows, 100GB data
- Heavy UPDATE traffic: 1M updates/day
- Old VACUUM configuration (default)

After 6 months:
- 180M dead tuples accumulated
- Disk shows 200GB (2x original data size)
- Indexes also swollen with dead entries
```

**Diagram:**
```
Day 1:     [Live Data: 100GB] [Dead: 0]
Month 1:   [Live Data: 100GB] [Dead: 10GB]
Month 3:   [Live Data: 100GB] [Dead: 50GB]
Month 6:   [Live Data: 100GB] [Dead: 100GB] ← DISK FULL
```

## How to Check Dead Tuples

```sql
-- See dead tuple count per table
SELECT schemaname, tablename, n_dead_tup, n_live_tup,
       ROUND(100 * n_dead_tup / (n_live_tup + n_dead_tup), 1) as dead_percent
FROM pg_stat_user_tables
ORDER BY n_dead_tup DESC;

-- Example output:
-- users      | 1000000 live | 500000 dead (33% dead) ← High! Needs VACUUM
-- orders     | 5000000 live | 100000 dead (2% dead)  ← Normal
-- products   | 100000 live  | 2000 dead   (2% dead)  ← OK

-- Disk space used
SELECT schemaname, tablename, pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)) as size
FROM pg_tables
WHERE schemaname NOT IN ('pg_catalog', 'information_schema')
ORDER BY pg_total_relation_size(schemaname||'.'||tablename) DESC
LIMIT 10;
```

## Auto-VACUUM Configuration

**Default (often too aggressive or not aggressive enough):**
```ini
# postgresql.conf

# Autovacuum process wakes every 10 seconds
autovacuum_naptime = 10s

# Vacuum triggers when:
# dead_tuples > autovacuum_vacuum_threshold +
#              (live_tuples * autovacuum_vacuum_scale_factor)

autovacuum_vacuum_threshold = 50       # Base: 50 dead tuples
autovacuum_vacuum_scale_factor = 0.1   # Plus: 10% of live tuples

# Example: 100M row table needs 10M+ dead tuples before VACUUM
# That's tons of garbage before cleanup!
```

## Recommended Production Configuration

```ini
# postgresql.conf

# More aggressive autovacuum
autovacuum = on
autovacuum_naptime = 10s            # Check every 10 seconds
autovacuum_vacuum_threshold = 1000  # Lower threshold
autovacuum_vacuum_scale_factor = 0.05  # Lower % needed
autovacuum_analyze_threshold = 500
autovacuum_analyze_scale_factor = 0.02

# Prevent autovacuum from overwhelming server
autovacuum_max_workers = 4          # Parallel vacuum workers
autovacuum_work_mem = 256MB         # Memory per worker

# Don't let long transactions block VACUUM
idle_in_transaction_session_timeout = '10 min'

# For very large, busy tables: per-table override
ALTER TABLE users SET (
    autovacuum_vacuum_scale_factor = 0.01,   # Even more aggressive
    autovacuum_vacuum_threshold = 10000,
    autovacuum_analyze_scale_factor = 0.005
);

# Aggressive index cleanup
autovacuum_vacuum_cost_delay = 10ms     # Delay between maintenance
autovacuum_vacuum_cost_limit = 10000    # Budget per iteration
```

## Manual VACUUM Strategy

```sql
-- During low-traffic window (e.g., 2 AM)
BEGIN TRANSACTION;
VACUUM ANALYZE users;
VACUUM ANALYZE orders;
VACUUM ANALYZE products;
COMMIT;
-- Typical time: 5-30 minutes for 100GB

-- Monitor progress
SELECT pid, query, query_start, state
FROM pg_stat_activity
WHERE query LIKE '%VACUUM%';

-- Aggressive cleanup (if needed)
-- Run monthly or quarterly during maintenance window
VACUUM FULL ANALYZE users;  -- Locks table, takes 30+ minutes
```

## Monitoring Dashboard

```sql
-- Create monitoring query
SELECT
    schemaname,
    tablename,
    n_live_tup as live_rows,
    n_dead_tup as dead_rows,
    ROUND(100.0 * n_dead_tup / NULLIF(n_live_tup + n_dead_tup, 0), 2) as dead_percent,
    last_vacuum,
    last_autovacuum,
    CASE
        WHEN n_dead_tup > (n_live_tup * 0.1) THEN 'HIGH'
        WHEN n_dead_tup > (n_live_tup * 0.05) THEN 'MEDIUM'
        ELSE 'LOW'
    END as urgency,
    pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)) as table_size
FROM pg_stat_user_tables
WHERE n_live_tup > 100000  -- Only large tables
ORDER BY n_dead_tup DESC;
```

## Recovery: Disk is 100% Full

```sql
-- Emergency cleanup (may block queries)

-- 1. VACUUM FULL biggest tables
VACUUM FULL ANALYZE public.users;
VACUUM FULL ANALYZE public.orders;

-- 2. Reindex to reclaim space
REINDEX INDEX idx_users_email;
REINDEX TABLE users;

-- 3. Check freed space
SELECT pg_size_pretty(pg_database_size('mydb'));

-- 4. Consider archiving old data
DELETE FROM logs WHERE created_at < now() - interval '1 year';
VACUUM ANALYZE logs;
```
