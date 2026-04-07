## Range Partitioned Orders Table

```sql
-- Create parent partitioned table
CREATE TABLE orders (
  id BIGSERIAL PRIMARY KEY,
  user_id UUID NOT NULL,
  amount DECIMAL(10,2) NOT NULL,
  status VARCHAR NOT NULL,
  created_at TIMESTAMP NOT NULL,
  updated_at TIMESTAMP NOT NULL
) PARTITION BY RANGE (YEAR(created_at), MONTH(created_at));

-- Create 2024 partitions
CREATE TABLE orders_2024_01 PARTITION OF orders
  FOR VALUES FROM (2024, 1) TO (2024, 2);

CREATE TABLE orders_2024_02 PARTITION OF orders
  FOR VALUES FROM (2024, 2) TO (2024, 3);

CREATE TABLE orders_2024_03 PARTITION OF orders
  FOR VALUES FROM (2024, 3) TO (2024, 4);

CREATE TABLE orders_2024_04 PARTITION OF orders
  FOR VALUES FROM (2024, 4) TO (2024, 5);

-- Indexes on each partition
CREATE INDEX idx_orders_2024_01_user_created
  ON orders_2024_01(user_id, created_at DESC);

CREATE INDEX idx_orders_2024_02_user_created
  ON orders_2024_02(user_id, created_at DESC);

CREATE INDEX idx_orders_2024_03_user_created
  ON orders_2024_03(user_id, created_at DESC);

CREATE INDEX idx_orders_2024_04_user_created
  ON orders_2024_04(user_id, created_at DESC);

-- Constraints per partition
ALTER TABLE orders_2024_01
  ADD CONSTRAINT check_status_01
    CHECK (status IN ('pending', 'completed', 'failed'));
```

## Querying Partitioned Table

```sql
-- Auto selects correct partition(s)
SELECT * FROM orders
WHERE user_id = '550e8400-e29b-41d4-a716-446655440000'
  AND created_at >= '2024-04-01'::timestamp
ORDER BY created_at DESC
LIMIT 10;

-- Partition pruning shows only 1 partition scanned
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM orders
WHERE created_at >= '2024-04-01' AND created_at < '2024-05-01';
-- Seq Scan on orders_2024_04 (or relevant partition)

-- Cross-partition query scans multiple
EXPLAIN (ANALYZE)
SELECT * FROM orders
WHERE created_at >= '2024-01-01' AND created_at < '2024-03-01';
-- Seq Scan on orders_2024_01 + orders_2024_02
```

## Automatic Partition Creation

```sql
-- Function to create next month's partition
CREATE OR REPLACE FUNCTION create_next_orders_partition()
RETURNS void AS $$
DECLARE
  next_year int;
  next_month int;
  partition_name text;
  from_values text;
  to_values text;
BEGIN
  -- Calculate next month
  SELECT
    EXTRACT(YEAR FROM CURRENT_DATE + INTERVAL '1 month')::int,
    EXTRACT(MONTH FROM CURRENT_DATE + INTERVAL '1 month')::int
  INTO next_year, next_month;

  partition_name := format('orders_%s_%s',
    next_year, LPAD(next_month::text, 2, '0'));

  from_values := format('(%s, %s)', next_year, next_month);
  to_values := format('(%s, %s)',
    CASE WHEN next_month = 12 THEN next_year + 1 ELSE next_year END,
    CASE WHEN next_month = 12 THEN 1 ELSE next_month + 1 END);

  EXECUTE format(
    'CREATE TABLE IF NOT EXISTS %I PARTITION OF orders
     FOR VALUES FROM %s TO %s',
    partition_name, from_values, to_values
  );

  -- Create index
  EXECUTE format(
    'CREATE INDEX %I ON %I(user_id, created_at DESC)',
    'idx_' || partition_name || '_user_created',
    partition_name
  );

  RAISE NOTICE 'Created partition: %', partition_name;
END;
$$ LANGUAGE plpgsql;

-- Trigger on first of each month
SELECT cron.schedule(
  'create_orders_partitions',
  '0 0 1 * *',  -- 00:00 on 1st of month
  'SELECT create_next_orders_partition();'
);
```

## Archiving Old Partitions

```sql
-- Move old partition to archive table (keep for compliance)
CREATE TABLE orders_archive (LIKE orders);

-- Move data
INSERT INTO orders_archive SELECT * FROM orders_2023_01;

-- Drop partition (fast - no data copy)
DROP TABLE orders_2023_01;

-- Or compress old partitions (PostgreSQL 14+)
ALTER TABLE orders_2023_06
  SET (fillfactor = 50);  -- Lower for archival partitions

VACUUM FULL orders_2023_06;  -- Compress
```

## Verify Partitioning

```sql
-- Check partition layout
SELECT
  schemaname,
  tablename,
  pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)) as size
FROM pg_tables
WHERE tablename LIKE 'orders%'
ORDER BY tablename;

-- Output:
-- orders_2024_01 | 120 MB
-- orders_2024_02 | 125 MB
-- orders_2024_03 | 118 MB
-- orders_2024_04 | 95 MB (current month, still growing)

-- Index sizes
SELECT
  schemaname,
  tablename,
  indexname,
  pg_size_pretty(pg_relation_size(schemaname||'.'||indexname)) as size
FROM pg_indexes
WHERE tablename LIKE 'orders%'
ORDER BY tablename, indexname;
```

## Performance Before/After

```
Query: Get user's last 10 orders from this month
WHERE user_id = ? AND created_at >= '2024-04-01'

BEFORE PARTITIONING:
- Table size: 500MB, Index: 200MB
- Scan: Full index on 500M rows
- Time: 2000ms
- Load: High (cache misses)

AFTER PARTITIONING:
- Partition size: 10MB, Index: 4MB
- Scan: Index on 1 partition (10M rows)
- Time: 100ms (20x faster)
- Load: Low (fits in cache)
```
