## When to Use Each Approach

### Soft Deletes (Pros & Cons)

**Pros:**
- Easy undo (restore deleted data)
- Maintain referential integrity easily
- Fast deletes (just update timestamp)
- Preserves audit history

**Cons:**
- Every query needs `WHERE deleted_at IS NULL`
- Indexes become bloated
- Disk usage grows indefinitely
- Causes performance degradation over time

**Use when:**
- Undo/restore is critical feature
- Legal/compliance requires audit trail
- Deletes are rare (< 10%)
- Dataset is small (< 1GB)

## Hard Deletes with Audit Table

Store audit trail separately:

```sql
-- Main table (clean, no deleted rows)
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    email TEXT,
    name TEXT,
    created_at TIMESTAMP
);

-- Audit table (immutable log)
CREATE TABLE user_audit (
    id SERIAL PRIMARY KEY,
    user_id INT,
    action TEXT,  -- 'insert', 'update', 'delete'
    old_data JSONB,
    new_data JSONB,
    changed_by TEXT,
    changed_at TIMESTAMP DEFAULT now()
);

-- Trigger to log deletes
CREATE OR REPLACE FUNCTION audit_user_delete()
RETURNS TRIGGER AS $$
BEGIN
    INSERT INTO user_audit (user_id, action, old_data, changed_at)
    VALUES (OLD.id, 'delete', row_to_json(OLD), now());
    RETURN OLD;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trigger_audit_delete
BEFORE DELETE ON users
FOR EACH ROW
EXECUTE FUNCTION audit_user_delete();

-- Hard delete
DELETE FROM users WHERE id = 42;

-- Check audit trail
SELECT * FROM user_audit WHERE user_id = 42;

-- Restore (if needed)
INSERT INTO users SELECT * FROM user_audit WHERE user_id = 42;

-- Benefits:
-- - Queries are fast (no dead rows)
-- - Disk not bloated
-- - Full audit trail in separate table
-- - Can archive audit table later
```

## Event Sourcing Pattern

Event-based immutable log:

```sql
-- Events: All state changes
CREATE TABLE user_events (
    id SERIAL PRIMARY KEY,
    user_id INT,
    event_type TEXT,  -- 'created', 'email_changed', 'deleted'
    data JSONB,
    version INT,      -- Event version (for ordering)
    created_at TIMESTAMP DEFAULT now()
);

-- Current state: Derived from events
CREATE MATERIALIZED VIEW user_state AS
SELECT
    user_id,
    (data->>'email') as email,
    (data->>'name') as name,
    MAX(version) as current_version
FROM (
    SELECT DISTINCT ON (user_id)
        user_id,
        data,
        version,
        event_type
    FROM user_events
    WHERE event_type != 'deleted'
    ORDER BY user_id, version DESC
) AS latest
GROUP BY user_id;

-- Track event
INSERT INTO user_events VALUES (DEFAULT, 42, 'created', '{"email":"alice@ex.com","name":"Alice"}', 1, now());
INSERT INTO user_events VALUES (DEFAULT, 42, 'email_changed', '{"email":"alice.smith@ex.com"}', 2, now());
INSERT INTO user_events VALUES (DEFAULT, 42, 'deleted', NULL, 3, now());

-- Current state (user deleted)
SELECT * FROM user_state WHERE user_id = 42;
-- Returns nothing (no row with event_type != 'deleted')

-- Replay history
SELECT * FROM user_events WHERE user_id = 42 ORDER BY version;

-- Undelete
INSERT INTO user_events VALUES (DEFAULT, 42, 'restored', NULL, 4, now());
-- User appears again in user_state

-- Benefits:
-- - Complete audit trail with timestamps
-- - Can replay to any point in time
-- - Undo/redo easily
-- - Events table is append-only, optimized for inserts
-- Drawbacks:
-- - Complex queries (require joins/aggregation)
-- - Storage: Multiple rows per entity
```

## Hybrid: Active Table + Archive

Best of both worlds:

```sql
-- Active table (only current data, no deleted_at)
CREATE TABLE accounts (
    id SERIAL PRIMARY KEY,
    company_name TEXT,
    status TEXT,  -- 'active', 'paused', 'trial'
    updated_at TIMESTAMP
);

-- Archive table (historical, soft-deleted)
CREATE TABLE accounts_archive (
    id SERIAL PRIMARY KEY,
    company_name TEXT,
    status TEXT,
    deleted_at TIMESTAMP,
    deleted_reason TEXT,
    archived_at TIMESTAMP DEFAULT now()
);

-- Function: Move to archive
CREATE OR REPLACE FUNCTION archive_account(account_id INT, reason TEXT)
RETURNS void AS $$
BEGIN
    INSERT INTO accounts_archive
    SELECT id, company_name, status, now(), reason, now()
    FROM accounts WHERE id = account_id;

    DELETE FROM accounts WHERE id = account_id;
END;
$$ LANGUAGE plpgsql;

-- Usage
SELECT archive_account(42, 'Customer requested');

-- Benefits:
-- - Active queries are fast (no deleted rows)
-- - Archive queries can restore if needed
-- - Disk not bloated (archive is separate)
-- - Can compress/cold-storage archive table

-- Restore from archive
INSERT INTO accounts
SELECT id, company_name, status FROM accounts_archive
WHERE id = 42;
DELETE FROM accounts_archive WHERE id = 42;
```

## Temporal Tables (PostgreSQL 12+)

Track all versions:

```sql
-- Modern approach: JSON-based versioning
CREATE TABLE accounts (
    id SERIAL PRIMARY KEY,
    company_name TEXT,
    status TEXT,
    valid_from TIMESTAMP DEFAULT now(),
    valid_to TIMESTAMP DEFAULT NULL
);

-- Function: Insert new version
CREATE OR REPLACE FUNCTION update_account_temporal(
    account_id INT,
    new_company_name TEXT,
    new_status TEXT
)
RETURNS void AS $$
BEGIN
    -- Close old version
    UPDATE accounts
    SET valid_to = now()
    WHERE id = account_id AND valid_to IS NULL;

    -- Insert new version
    INSERT INTO accounts (id, company_name, status, valid_from)
    VALUES (account_id, new_company_name, new_status, now());
END;
$$ LANGUAGE plpgsql;

-- View: Current state
CREATE VIEW accounts_current AS
SELECT * FROM accounts WHERE valid_to IS NULL;

-- View: Historical versions
CREATE VIEW accounts_history AS
SELECT id, company_name, status, valid_from, valid_to FROM accounts;

-- Query current
SELECT * FROM accounts_current;

-- Query as of date
SELECT * FROM accounts_history
WHERE id = 42 AND valid_from <= '2026-01-01' AND (valid_to IS NULL OR valid_to > '2026-01-01');
```

## Decision Matrix

| Approach | Restore | Audit | Query Speed | Storage | Complexity |
|----------|---------|-------|------------|---------|-----------|
| **Soft Delete** | Easy | Basic | Slow (bloats) | High | Very Low |
| **Hard Delete + Audit** | Possible | Full | Fast | Low | Low |
| **Event Sourcing** | Easy | Full | Medium | High | High |
| **Archive Table** | Easy | Full | Fast | Low-Med | Low |
| **Temporal** | Easy | Full | Medium | High | Medium |

**Recommendation:**
1. **Start with:** Soft deletes (simplest)
2. **When bloats:** Add partial index
3. **If archive needed:** Migrate to archive table
4. **For audit trail:** Add event logging separately
5. **If complex history:** Event sourcing or temporal tables
