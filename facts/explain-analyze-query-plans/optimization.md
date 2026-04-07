## Common Fixes for Slow Queries

### 1. Add Missing Indexes

**Problem:** Seq Scan shows 30s on 100M rows
```sql
EXPLAIN ANALYZE
SELECT * FROM orders WHERE customer_id = 123 AND status = 'shipped';
-- Seq Scan on orders (cost=0.00..2500000.00 rows=1000)
```

**Fix:** Composite index on filter columns
```sql
CREATE INDEX idx_orders_customer_status
ON orders(customer_id, status);

-- Now uses index
EXPLAIN ANALYZE
SELECT * FROM orders WHERE customer_id = 123 AND status = 'shipped';
-- Index Scan using idx_orders_customer_status (rows=1000)
```

### 2. Refresh Statistics

```sql
-- If estimated rows >> actual rows
ANALYZE users;  -- Full table
ANALYZE users(email);  -- Single column

-- Detailed stats for better planning
ALTER TABLE users ALTER COLUMN status SET STATISTICS 100;
ANALYZE users;
```

### 3. Rewrite Query Structure

**Nested Loop problem:**
```sql
-- BAD: Nested loop, 100M × 1M = 100T operations
EXPLAIN ANALYZE
SELECT o.id, c.name
FROM orders o
JOIN customers c ON o.customer_id = c.id
WHERE o.created_at > now() - interval '30 days';
-- Nested Loop with Filter rows=1000000
```

**Better: Add index to inner table**
```sql
CREATE INDEX idx_customers_id ON customers(id);

-- Now Hash Join or Merge Join (much faster)
EXPLAIN ANALYZE
SELECT o.id, c.name
FROM orders o
JOIN customers c ON o.customer_id = c.id
WHERE o.created_at > now() - interval '30 days';
-- Hash Join (actual time=...rows=50000)
```

### 4. Partial Indexes for Filtered Queries

```sql
-- If most queries filter on status='active'
CREATE INDEX idx_orders_active
ON orders(customer_id)
WHERE status = 'active';
-- Index size: 10GB (only active rows) vs 50GB (full index)

EXPLAIN ANALYZE
SELECT * FROM orders
WHERE customer_id = 123 AND status = 'active';
-- Index Only Scan (best case)
```

### 5. Cover Columns in Index (Index-Only Scan)

```sql
-- Need to fetch: id, email, status
CREATE INDEX idx_users_status_covered
ON users(status) INCLUDE (id, email);

EXPLAIN ANALYZE
SELECT id, email FROM users WHERE status = 'active';
-- Index Only Scan (doesn't touch table)
```

### 6. Query Rewriting: Push Conditions Down

**Bad: Filter after join**
```sql
EXPLAIN ANALYZE
SELECT * FROM (
  SELECT o.*, c.name FROM orders o
  JOIN customers c ON o.customer_id = c.id
) sub
WHERE sub.status = 'shipped' LIMIT 100;
-- Joins 100M rows, then filters
```

**Good: Filter before join**
```sql
EXPLAIN ANALYZE
SELECT o.*, c.name
FROM orders o
JOIN customers c ON o.customer_id = c.id
WHERE o.status = 'shipped'
LIMIT 100;
-- Filters first, joins fewer rows
```

### 7. Denormalization for Expensive Joins

```sql
-- Before: 3-table join on every query
EXPLAIN ANALYZE
SELECT o.id, c.name, i.total
FROM orders o
JOIN customers c ON o.customer_id = c.id
JOIN invoices i ON o.id = i.order_id
WHERE o.created_at > now() - interval '30 days';
-- 3 joins on 100M orders = slow

-- After: Denormalize frequently-accessed data
ALTER TABLE orders ADD COLUMN customer_name TEXT;
CREATE INDEX idx_orders_created ON orders(created_at);

SELECT o.id, o.customer_name, i.total
FROM orders o
JOIN invoices i ON o.id = i.order_id
WHERE o.created_at > now() - interval '30 days';
-- 2 joins instead of 3
```

## Real-World Example: 30s → 200ms

**Original slow query:**
```sql
EXPLAIN ANALYZE
SELECT DISTINCT u.id, u.email, COUNT(o.id) as order_count
FROM users u
LEFT JOIN orders o ON u.id = o.user_id
WHERE u.created_at > '2025-01-01'
AND u.status = 'active'
AND o.total > 100
GROUP BY u.id, u.email
HAVING COUNT(o.id) > 5
ORDER BY order_count DESC
LIMIT 100;
-- Seq Scan on users (30 seconds)
```

**Optimized:**
```sql
-- 1. Add indexes on filters
CREATE INDEX idx_users_status_created
ON users(status, created_at);

-- 2. Add index on join + filter
CREATE INDEX idx_orders_user_total
ON orders(user_id, total);

-- 3. Rewrite to reduce early filtering
EXPLAIN ANALYZE
SELECT u.id, u.email, COUNT(o.id) as order_count
FROM users u
INNER JOIN (
  SELECT user_id, COUNT(*) as cnt
  FROM orders
  WHERE total > 100
  GROUP BY user_id
  HAVING COUNT(*) > 5
) o ON u.id = o.user_id
WHERE u.created_at > '2025-01-01'
AND u.status = 'active'
ORDER BY order_count DESC
LIMIT 100;
-- Hash Join (200ms) - 150x faster
```

## Performance Testing Pattern

```bash
# Run 3 times to warm cache
psql -c "EXPLAIN ANALYZE <query>" > /dev/null
psql -c "EXPLAIN ANALYZE <query>" > /dev/null
psql -c "EXPLAIN ANALYZE <query>" > perf_after.txt

# Compare with baseline
diff perf_before.txt perf_after.txt
```
