## The Problem: Soft Deletes Scaling

```sql
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    email TEXT,
    deleted_at TIMESTAMP  -- NULL = active, non-NULL = deleted
);

-- Every query must filter deleted_at IS NULL
SELECT * FROM users WHERE deleted_at IS NULL AND email LIKE '%@example.com%';
-- On 100M rows (80M deleted): Scans 80M dead rows to find 20M active

-- After 2 years: Disk = 4x actual data
-- Query speed: 10x slower than it should be
```

## Solution 1: Partial Index (Simplest)

Index only active rows:

```sql
-- Create index on active records only
CREATE INDEX idx_users_active_email ON users(email)
WHERE deleted_at IS NULL;

-- Query uses small index
SELECT * FROM users
WHERE deleted_at IS NULL AND email = 'alice@example.com';
-- Index Scan (only 20M rows scanned, not 100M)

-- Benefits:
-- - Index size: ~20% of full index
-- - Faster queries
-- - Only pays for active data
-- - No data movement required
```

## Solution 2: Table Partitioning

Separate active/deleted into different partitions:

```sql
-- Create partitioned table
CREATE TABLE users (
    id SERIAL,
    email TEXT,
    deleted_at TIMESTAMP
) PARTITION BY RANGE (deleted_at);

-- Active partition (NULL values)
CREATE TABLE users_active PARTITION OF users
    FOR VALUES FROM (UNBOUNDED) TO (NULL);

-- Deleted partition (2024)
CREATE TABLE users_deleted_2024 PARTITION OF users
    FOR VALUES FROM ('2024-01-01') TO ('2025-01-01');

-- Deleted partition (2025)
CREATE TABLE users_deleted_2025 PARTITION OF users
    FOR VALUES FROM ('2025-01-01') TO ('2026-01-01');

-- Indexes on partitions
CREATE INDEX idx_users_active_email ON users_active(email);
CREATE INDEX idx_users_deleted_2024_id ON users_deleted_2024(id);

-- Queries naturally use partition pruning
SELECT * FROM users WHERE deleted_at IS NULL;
-- Scan ONLY users_active table (much smaller)

-- Delete old partitions easily
DROP TABLE users_deleted_2023;
-- Instant (no scanning), frees disk immediately
```

## Solution 3: Archival Strategy

Move deleted data to archive table:

```sql
-- Main table (only active users)
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    email TEXT
    -- No deleted_at column!
);

-- Archive table (stores deleted users)
CREATE TABLE users_archive (
    id SERIAL,
    email TEXT,
    deleted_at TIMESTAMP,
    archived_at TIMESTAMP DEFAULT now()
);

-- Hard delete from main, soft delete to archive
DELETE FROM users WHERE email = 'bob@example.com';
-- Or move to archive:

INSERT INTO users_archive (id, email, deleted_at)
SELECT id, email, now() FROM users WHERE condition;

DELETE FROM users WHERE condition;

-- Benefits:
-- - Active table is small and fast
-- - Queries don't need WHERE deleted_at IS NULL
-- - Archive is compressed/cold storage
-- - Restore deleted user: MOVE from archive back to main
```

## Real Example: SaaS Platform

```sql
-- Phase 1: Create archive table
CREATE TABLE accounts_archive (
    id BIGINT,
    company_name TEXT,
    subscription_plan TEXT,
    deleted_at TIMESTAMP,
    archived_at TIMESTAMP DEFAULT now()
);

-- Phase 2: Backfill active accounts to main table (remove deleted ones)
CREATE TABLE accounts_new (
    id BIGINT PRIMARY KEY,
    company_name TEXT,
    subscription_plan TEXT
);

INSERT INTO accounts_new
SELECT id, company_name, subscription_plan
FROM accounts WHERE deleted_at IS NULL;

-- Phase 3: Move deleted records to archive
INSERT INTO accounts_archive
SELECT id, company_name, subscription_plan, deleted_at, now()
FROM accounts WHERE deleted_at IS NOT NULL;

-- Phase 4: Switch tables
DROP TABLE accounts;
ALTER TABLE accounts_new RENAME TO accounts;
CREATE INDEX idx_accounts_name ON accounts(company_name);

-- Result: Active table is 20% of original size, 5x faster queries
```

## Solution 4: Time-Series Delete

Mark delete timestamp, query only recent data:

```sql
-- For data that's time-sensitive (logs, events)
CREATE TABLE events (
    id SERIAL PRIMARY KEY,
    user_id INT,
    event_type TEXT,
    created_at TIMESTAMP,
    deleted_at TIMESTAMP,
    year INT GENERATED ALWAYS AS (EXTRACT(YEAR FROM created_at)) STORED
);

-- Partition by year
CREATE TABLE events_2024 PARTITION OF events
    FOR VALUES FROM ('2024') TO ('2025');

CREATE TABLE events_2025 PARTITION OF events
    FOR VALUES FROM ('2025') TO ('2026');

-- Soft delete within year, hard delete after retention
DELETE FROM events_2024 WHERE deleted_at < now() - interval '90 days';
-- After 90 days, hard delete (free disk immediately)

-- Queries naturally use partition pruning
SELECT * FROM events
WHERE created_at > now() - interval '30 days'
  AND deleted_at IS NULL;
-- Only scans events_2025 partition
```

## Solution 5: Undelete with Audit Trail

Use event sourcing pattern:

```sql
-- Events table (immutable, append-only)
CREATE TABLE user_events (
    id SERIAL PRIMARY KEY,
    user_id INT,
    event_type TEXT,  -- 'created', 'updated', 'deleted', 'undeleted'
    data JSONB,
    created_at TIMESTAMP DEFAULT now()
);

-- View: Current state of users
CREATE VIEW users AS
SELECT
    user_id as id,
    (data->>'email') as email,
    (data->>'name') as name,
    MAX(created_at) as updated_at
FROM user_events
WHERE event_type IN ('created', 'updated')
  AND NOT EXISTS (
    SELECT 1 FROM user_events e2
    WHERE e2.user_id = user_events.user_id
      AND e2.event_type = 'deleted'
      AND e2.created_at > user_events.created_at
  )
GROUP BY user_id;

-- Insert user
INSERT INTO user_events VALUES (DEFAULT, 42, 'created', '{"email":"alice@ex.com","name":"Alice"}', now());

-- Soft delete
INSERT INTO user_events VALUES (DEFAULT, 42, 'deleted', NULL, now());

-- Undelete
INSERT INTO user_events VALUES (DEFAULT, 42, 'undeleted', NULL, now());

-- View shows user again (no hard delete)
SELECT * FROM users WHERE id = 42;

-- Full audit trail
SELECT * FROM user_events WHERE user_id = 42 ORDER BY created_at;
-- Shows: created -> deleted -> undeleted
```

## Performance Comparison

```
SCENARIO: 100M users, 80% soft-deleted

Strategy: No optimization
- Query: SELECT * WHERE deleted_at IS NULL
- Time: 5000ms (scans 100M rows, filters 80M)
- Storage: 100GB
- Index size: 40GB

Strategy: Partial Index
- Query: SELECT * WHERE deleted_at IS NULL
- Time: 100ms (scans 20M rows via small index)
- Storage: 100GB (no change)
- Index size: 8GB (80% savings!)

Strategy: Partitioning
- Query: SELECT * WHERE deleted_at IS NULL
- Time: 50ms (partition pruning, scans 20M)
- Storage: 100GB
- Index size: 8GB
- Hard delete: Instant (DROP partition)

Strategy: Archival
- Query: SELECT * (no filter!)
- Time: 20ms (scans 20M)
- Storage: 20GB (main) + 80GB (cold archive)
- Index size: 4GB
- Performance improvement: 250x!
```

## Migration Path

For existing soft-delete tables:

```sql
-- Step 1: Create partial index (no downtime, immediate benefit)
CREATE INDEX idx_users_active ON users(email) WHERE deleted_at IS NULL;

-- Step 2: Create archive table
CREATE TABLE users_archive AS
SELECT * FROM users WHERE deleted_at IS NOT NULL;

-- Step 3: During maintenance window, move old deletes to archive
DELETE FROM users WHERE deleted_at < now() - interval '1 year';
-- (Assuming > 1 year old is safe to remove)

-- Step 4: Add trigger for future soft deletes
CREATE OR REPLACE FUNCTION archive_deleted_users()
RETURNS TRIGGER AS $$
BEGIN
    IF NEW.deleted_at IS NOT NULL AND OLD.deleted_at IS NULL THEN
        -- User just soft-deleted, archive after 1 year
        INSERT INTO users_archive VALUES (NEW.*);
    END IF;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trigger_archive_users
AFTER UPDATE ON users
FOR EACH ROW
EXECUTE FUNCTION archive_deleted_users();
```
