## JSONB Indexing Pitfalls

### Pitfall 1: No Index on Frequently-Queried Fields

```sql
-- BAD: Full table scan
SELECT * FROM users
WHERE metadata->>'status' = 'active'
  AND metadata->>'tier' = 'premium';
-- Sequential scan on 100M rows = 30+ seconds

-- FIX: Add specific indexes
CREATE INDEX idx_users_status ON users USING btree((metadata->>'status'));
CREATE INDEX idx_users_tier ON users USING btree((metadata->>'tier'));
-- Now both conditions use indexes, < 100ms
```

### Pitfall 2: Text-to-Integer Conversion in Query

```sql
-- BAD: Type conversion in WHERE clause prevents full index use
SELECT * FROM users WHERE (metadata->>'age')::INTEGER > 30;
-- Index exists but PostgreSQL must convert all values (slow)

-- BETTER: Use expression index
CREATE INDEX idx_users_age ON users USING btree(((metadata->>'age')::INTEGER));
-- Index stores already-converted values (fast)

-- Or cast the comparison value
SELECT * FROM users WHERE metadata->>'age' > '30';
-- String comparison (OK if stored as strings)
```

### Pitfall 3: Nested Object Queries Without Index

```sql
-- BAD: Accessing nested fields without index
SELECT * FROM orders
WHERE metadata->'shipping_address'->>'city' = 'NYC';
-- Sequential scan

-- GOOD: Create index on nested field
CREATE INDEX idx_orders_city ON orders USING btree((metadata->'shipping_address'->>'city'));
-- Now uses index (fast)
```

### Pitfall 4: GIN Index on Everything (Overkill)

```sql
-- BAD: Large generic GIN index on whole JSONB
CREATE INDEX idx_users_metadata_gin ON users USING gin(metadata);
-- Index size: 500GB (for 100GB data)
-- Slows down inserts/updates
-- Not much faster than specific indexes

-- BETTER: Specific indexes on hot fields
CREATE INDEX idx_users_status ON users USING btree((metadata->>'status'));
CREATE INDEX idx_users_tags ON users USING gin((metadata->'tags'));
-- Index size: 5GB (focused)
-- Faster queries AND faster writes
```

### Pitfall 5: Wrong Operator for Type

```sql
-- BAD: JSON value is integer, comparing as string
INSERT INTO users (metadata) VALUES ('{"count": 42}'::jsonb);

SELECT * FROM users WHERE metadata->>'count' = '42';  -- Works but slow
-- String comparison, index not used effectively

-- GOOD: Cast to correct type
SELECT * FROM users WHERE (metadata->>'count')::INTEGER = 42;
-- Can use btree index on integer values

-- BETTER: Use expression index
CREATE INDEX idx_users_count ON users USING btree(((metadata->>'count')::INTEGER));
```

### Pitfall 6: Array Containment Not Indexed

```sql
-- BAD: Array search without GIN index
SELECT * FROM users WHERE metadata->'tags' @> '"vip"'::jsonb;
-- Slow on large tables

-- GOOD: GIN index for array containment
CREATE INDEX idx_users_tags ON users USING gin((metadata->'tags'));
-- Now array queries are fast
```

### Pitfall 7: Not Using Correct Operators

```sql
-- Know the operators
-- @ > (contains): Check if left contains right
-- <- (contained by): Check if left is contained in right
-- ? (key exists): Check if key exists in object
-- ?| (any key exists): Check if any keys exist
-- ?& (all keys exist): Check if all keys exist

-- Examples:
SELECT * FROM users WHERE metadata @> '{"city": "NYC"}';  -- Contains city:NYC
SELECT * FROM users WHERE '{"city": "NYC"}' <@ metadata;  -- Same as above (reversed)
SELECT * FROM users WHERE metadata ? 'age';  -- age key exists
SELECT * FROM users WHERE metadata ?| ARRAY['age', 'name'];  -- Either key exists
SELECT * FROM users WHERE metadata ?& ARRAY['age', 'name'];  -- Both keys exist

-- Only @> and <- benefit from GIN index!
```

### Pitfall 8: Updating JSONB Performance

```sql
-- BAD: Full object replacement (slow on large JSONB)
UPDATE users SET metadata = '{"status": "active"}'::jsonb
WHERE id = 1;
-- Writes entire JSONB column

-- GOOD: Partial update with ||
UPDATE users SET metadata = metadata || '{"status": "active"}'::jsonb
WHERE id = 1;
-- PostgreSQL merges objects efficiently

-- Update nested field
UPDATE users SET metadata = jsonb_set(metadata, '{profile,age}', '30')
WHERE id = 1;

-- Remove field
UPDATE users SET metadata = metadata - 'temporary_field'
WHERE id = 1;
```

### Pitfall 9: JSONB Size Not Monitored

```sql
-- BAD: Storing large documents in JSONB
INSERT INTO logs (data) VALUES (
  '{"full_request_body": "... 100KB of data ...","full_response": "... 100KB ...", "entire_stack_trace": "... 50KB ..."}'::jsonb
);

-- Disk usage explodes, index becomes huge
SELECT pg_size_pretty(pg_total_relation_size('logs'));
-- 500GB for what should be 50GB

-- GOOD: Store only essential fields
INSERT INTO logs (data) VALUES (
  '{"error_code": 500, "message": "Database timeout", "user_id": 123, "timestamp": "2026-04-06T10:00:00Z"}'::jsonb
);

-- Archive full request/response separately
INSERT INTO log_archives (full_data) VALUES (original_request_object);
```

### Pitfall 10: No Validation of JSONB Structure

```sql
-- BAD: Accepting arbitrary JSON structure
INSERT INTO users (metadata) VALUES
('{"age": "30"}'),  -- String
('{"age": 30}'),    -- Integer
('{"city": "NYC"}'); -- Different schema

-- Now queries like (metadata->>'age')::INTEGER are inconsistent

-- GOOD: Validate structure with CHECK or trigger
CREATE OR REPLACE FUNCTION validate_user_metadata()
RETURNS TRIGGER AS $$
BEGIN
    -- Ensure required fields and types
    IF NOT (NEW.metadata ? 'status') THEN
        RAISE EXCEPTION 'metadata must contain status field';
    END IF;

    IF NOT ((NEW.metadata->>'status') IN ('active', 'inactive', 'pending')) THEN
        RAISE EXCEPTION 'status must be active, inactive, or pending';
    END IF;

    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trigger_validate_metadata
BEFORE INSERT OR UPDATE ON users
FOR EACH ROW
EXECUTE FUNCTION validate_user_metadata();
```

## Debugging JSONB Queries

```sql
-- See if index is being used
EXPLAIN ANALYZE
SELECT * FROM users WHERE (metadata->>'status')::TEXT = 'active';

-- Expected output with index:
-- Index Scan using idx_users_status on users (cost=0.42..8.44 rows=1000)
--   Index Cond: ((metadata->>'status') = 'active')

-- If you see "Seq Scan" or "Filter" instead, index isn't being used.
-- Common causes:
-- 1. Type mismatch (string vs integer)
-- 2. Missing index
-- 3. Query doesn't match index definition
```
