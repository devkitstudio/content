## Problem: Massive Tables

```sql
-- 500M rows in single table
-- Index on (user_id, created_at) = HUGE
-- Sequential scan still slow
-- Inserts slower (index maintenance)
-- Backups massive

SELECT * FROM orders WHERE user_id = 123 ORDER BY created_at DESC;
-- Still slow: indexes can't help enough with 500M rows
```

## Solution: Table Partitioning

Split 500M rows into smaller chunks. Query only relevant partition.

```
orders (500M rows)
├── orders_2024_01 (10M)
├── orders_2024_02 (10M)
├── orders_2024_03 (10M)
└── orders_2024_04 (10M)
```

Queries automatically read only needed partitions.

## Strategy 1: Range Partitioning (Recommended)

Partition by date range (typical for time-series data).

```sql
-- Create range-partitioned table
CREATE TABLE orders (
  id UUID PRIMARY KEY,
  user_id UUID NOT NULL,
  amount DECIMAL,
  created_at TIMESTAMP NOT NULL,
  status VARCHAR
) PARTITION BY RANGE (YEAR(created_at), MONTH(created_at));

-- Create partitions
CREATE TABLE orders_2024_01 PARTITION OF orders
  FOR VALUES FROM (2024, 1) TO (2024, 2);

CREATE TABLE orders_2024_02 PARTITION OF orders
  FOR VALUES FROM (2024, 2) TO (2024, 3);

-- Create indexes on partitions
CREATE INDEX idx_orders_2024_01_user ON orders_2024_01(user_id);
CREATE INDEX idx_orders_2024_02_user ON orders_2024_02(user_id);
```

**Benefits:**
- Old partitions can be archived (compressed)
- Inserts only go to current partition
- Queries on past data still work
- Drop old data easily

**Queries:**
```sql
-- Automatic partition pruning
SELECT * FROM orders WHERE created_at >= '2024-04-01';
-- Only reads 2024_04 partition

SELECT * FROM orders WHERE user_id = 123 AND created_at >= '2024-01-01';
-- Scans user_id index only on relevant partitions
```

## Strategy 2: List Partitioning

Partition by discrete values (regions, statuses).

```sql
CREATE TABLE orders PARTITION BY LIST (region);

CREATE TABLE orders_us PARTITION OF orders
  FOR VALUES IN ('US', 'CA', 'MX');

CREATE TABLE orders_eu PARTITION OF orders
  FOR VALUES IN ('UK', 'FR', 'DE', 'IT');

CREATE TABLE orders_asia PARTITION OF orders
  FOR VALUES IN ('JP', 'CN', 'SG', 'IN');
```

Good for:
- Regional data (GDPR compliance)
- Status-based (active, archived, deleted)
- Fixed categories

## Strategy 3: Hash Partitioning

Distribute evenly by hash (user_id, order_id).

```sql
CREATE TABLE orders PARTITION BY HASH (user_id);

CREATE TABLE orders_part_0 PARTITION OF orders
  FOR VALUES WITH (MODULUS 4, REMAINDER 0);

CREATE TABLE orders_part_1 PARTITION OF orders
  FOR VALUES WITH (MODULUS 4, REMAINDER 1);

CREATE TABLE orders_part_2 PARTITION OF orders
  FOR VALUES WITH (MODULUS 4, REMAINDER 2);

CREATE TABLE orders_part_3 PARTITION OF orders
  FOR VALUES WITH (MODULUS 4, REMAINDER 3);
```

Good for:
- Even distribution
- No time-based access
- Horizontal sharding prep

## Migration from Unpartitioned

```sql
-- Create new partitioned table
CREATE TABLE orders_new (
  id UUID PRIMARY KEY,
  user_id UUID,
  amount DECIMAL,
  created_at TIMESTAMP,
  status VARCHAR
) PARTITION BY RANGE (YEAR(created_at), MONTH(created_at));

-- Create partitions (multiple years)
CREATE TABLE orders_new_2023_01 PARTITION OF orders_new
  FOR VALUES FROM (2023, 1) TO (2023, 2);
-- ... create all partitions

-- Copy data (in batches to avoid locks)
INSERT INTO orders_new
SELECT * FROM orders
WHERE created_at >= '2023-01-01' AND created_at < '2023-02-01';

-- Add constraints
ALTER TABLE orders_new ADD CONSTRAINT orders_new_pk
  PRIMARY KEY (id, created_at);

-- Rename
ALTER TABLE orders RENAME TO orders_old;
ALTER TABLE orders_new RENAME TO orders;

-- Verify
SELECT * FROM orders WHERE created_at >= '2024-01-01';
```

## Performance Gains

```
Before partitioning:
SELECT * FROM orders WHERE created_at >= '2024-04-01'
→ Index scan on 500M rows
→ 2 seconds

After partitioning (range by month):
SELECT * FROM orders WHERE created_at >= '2024-04-01'
→ Partition pruning: only 2024_04, 2024_05, ... scanned
→ Index scan on 10M rows per partition
→ 100ms (20x faster)

Index size:
Before: 50GB per index
After: 1.2GB per month partition index = 14GB/year
```

## Maintenance

```sql
-- Auto-create next month's partition
CREATE OR REPLACE FUNCTION create_orders_partitions()
RETURNS void AS $$
DECLARE
  next_year int;
  next_month int;
BEGIN
  SELECT EXTRACT(YEAR FROM CURRENT_DATE + INTERVAL '1 month')::int,
         EXTRACT(MONTH FROM CURRENT_DATE + INTERVAL '1 month')::int
  INTO next_year, next_month;

  EXECUTE format('CREATE TABLE IF NOT EXISTS orders_%s_%s PARTITION OF orders
    FOR VALUES FROM (%L, %L) TO (%L, %L)',
    next_year, LPAD(next_month::text, 2, '0'),
    next_year, next_month,
    next_year, next_month + 1);
END;
$$ LANGUAGE plpgsql;

-- Run monthly
SELECT create_orders_partitions();

-- Archive old data (drop old partition)
DROP TABLE orders_2020_01;  -- Fast, only that partition
```
