## Safe Patterns for Common Operations

### Pattern 1: Renaming a Column

**Naive approach (downtime):**
```sql
-- Locks table, breaks existing queries
ALTER TABLE users RENAME COLUMN phone TO phone_number;
```

**Safe approach:**
```sql
-- Step 1: Add new column with data
ALTER TABLE users ADD COLUMN phone_number VARCHAR(20);
UPDATE users SET phone_number = phone WHERE phone IS NOT NULL;

-- Step 2: Application reads both columns
-- In code: use phone_number if not null, else phone

-- Step 3: Switch writes to new column
-- In code: write to phone_number only

-- Step 4: Remove old column (after monitoring)
ALTER TABLE users DROP COLUMN phone;
```

### Pattern 2: Changing Column Type (e.g., INT to BIGINT)

**Risky (rewrite):**
```sql
-- Locks entire table for the duration
ALTER TABLE orders ALTER COLUMN order_id TYPE BIGINT;
```

**Safe approach:**
```sql
-- Step 1: Add new column with new type
ALTER TABLE orders ADD COLUMN order_id_new BIGINT;

-- Step 2: Copy data in batches
DO $$
DECLARE
    batch_size INT := 50000;
    processed INT := 0;
BEGIN
    LOOP
        UPDATE orders SET order_id_new = order_id
        WHERE order_id_new IS NULL
        LIMIT batch_size;

        processed := processed + (SELECT CHANGES());
        IF processed = 0 THEN EXIT; END IF;

        PERFORM pg_sleep(1);  -- Brief pause
    END LOOP;
END $$;

-- Step 3: Swap columns
ALTER TABLE orders RENAME COLUMN order_id TO order_id_old;
ALTER TABLE orders RENAME COLUMN order_id_new TO order_id;

-- Step 4: Remove old column
ALTER TABLE orders DROP COLUMN order_id_old;
```

### Pattern 3: Adding Foreign Key Constraint

**Risky (validates all rows):**
```sql
-- Locks table while validating 50M rows
ALTER TABLE orders ADD CONSTRAINT fk_orders_users
FOREIGN KEY (user_id) REFERENCES users(id);
```

**Safe approach:**
```sql
-- Step 1: Add constraint as NOT VALID (instant)
ALTER TABLE orders ADD CONSTRAINT fk_orders_users
FOREIGN KEY (user_id) REFERENCES users(id) NOT VALID;

-- Step 2: Find and fix violating rows
SELECT user_id, COUNT(*) FROM orders
WHERE user_id NOT IN (SELECT id FROM users)
GROUP BY user_id;

-- Step 3: Validate (brief lock, only bad rows checked)
ALTER TABLE orders VALIDATE CONSTRAINT fk_orders_users;
```

### Pattern 4: Adding CHECK Constraint

```sql
-- Add constraint without validating existing data
ALTER TABLE users ADD CONSTRAINT age_positive
CHECK (age > 0) NOT VALID;

-- Fix violations asynchronously
UPDATE users SET age = 18 WHERE age <= 0;

-- Then validate
ALTER TABLE users VALIDATE CONSTRAINT age_positive;
```

### Pattern 5: Changing Default Value

**Simple (safe):**
```sql
-- Metadata-only change, no rewrite, instant
ALTER TABLE users ALTER COLUMN status SET DEFAULT 'active';
```

### Pattern 6: Adding GENERATED Column (PostgreSQL 12+)

```sql
-- Fast, doesn't rewrite table
ALTER TABLE orders ADD COLUMN total_with_tax NUMERIC
GENERATED ALWAYS AS (total * 1.1) STORED;

-- For older PostgreSQL: use computed column + trigger
CREATE OR REPLACE FUNCTION orders_update_total_with_tax()
RETURNS TRIGGER AS $$
BEGIN
    NEW.total_with_tax := NEW.total * 1.1;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trigger_orders_total_with_tax
BEFORE INSERT OR UPDATE ON orders
FOR EACH ROW
EXECUTE FUNCTION orders_update_total_with_tax();
```

## Checklist for Production Migrations

- [ ] Test migration on staging database (same size as production)
- [ ] Measure how long each step takes
- [ ] Identify if step locks the table (look for ExclusiveLock)
- [ ] Plan migration window or ensure steps are non-blocking
- [ ] Backup database before migration
- [ ] Create index CONCURRENTLY, not regular CREATE INDEX
- [ ] Use NOT VALID for constraints, validate separately
- [ ] Batch updates with sleeps to avoid overwhelming server
- [ ] Monitor pg_stat_activity during migration
- [ ] Have rollback plan (keep old column/constraint temporarily)
- [ ] Test application with new schema before removing old columns
- [ ] After migration, update table statistics: ANALYZE

## Performance Tips

```sql
-- For large updates, disable triggers temporarily
ALTER TABLE users DISABLE TRIGGER ALL;
UPDATE users SET ... WHERE ...;
ALTER TABLE users ENABLE TRIGGER ALL;

-- Use UNLOGGED tables for temporary work (faster, lost on crash)
CREATE UNLOGGED TABLE temp_users AS SELECT * FROM users;

-- Disable autovacuum during bulk operations
ALTER TABLE users SET (autovacuum_enabled = false);
-- ... do work ...
ALTER TABLE users SET (autovacuum_enabled = true);
VACUUM ANALYZE users;
```
