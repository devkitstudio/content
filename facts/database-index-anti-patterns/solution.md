## Anti-Pattern 1: Indexing Every Column

Every index is a **separate B-tree** that must be updated on every INSERT, UPDATE, DELETE. More indexes = slower writes.

```
Table: orders (1M rows)
- 0 indexes: INSERT = 0.5ms
- 3 indexes: INSERT = 1.5ms
- 10 indexes: INSERT = 5ms   ← 10x slower
- 20 indexes: INSERT = 12ms  ← production pain
```

**Rule**: Only index columns that appear in `WHERE`, `JOIN`, `ORDER BY` of your actual slow queries.

## Anti-Pattern 2: Low-Cardinality Index

Indexing a column with very few distinct values (e.g., `status ENUM('active','inactive')`) is almost always useless.

```sql
-- This index is USELESS: 50% of rows match, full scan is faster
CREATE INDEX idx_status ON users(status);
SELECT * FROM users WHERE status = 'active'; -- 500K of 1M rows

-- This index is USEFUL: very few rows match
CREATE INDEX idx_status ON users(status);
SELECT * FROM users WHERE status = 'suspended'; -- 50 of 1M rows
```

The query planner does the math: if the index returns >15-20% of all rows, a sequential scan is cheaper.

## Anti-Pattern 3: Wrong Column Order in Composite Index

```sql
-- You create this composite index:
CREATE INDEX idx_orders ON orders(status, created_at);

-- This query USES the index (leftmost prefix match):
SELECT * FROM orders WHERE status = 'paid' AND created_at > '2026-01-01';

-- This query IGNORES the index:
SELECT * FROM orders WHERE created_at > '2026-01-01';
-- ↑ Skips `status` (the first column), so the index tree can't be traversed
```

**Rule**: Put the most selective (highest cardinality) column FIRST in composite indexes. Think of it like a phone book: last name first, then first name.

## Anti-Pattern 4: Indexing Expressions Without Expression Index

```sql
-- Your index on `email` won't help this query:
CREATE INDEX idx_email ON users(email);
SELECT * FROM users WHERE LOWER(email) = 'john@example.com';
-- ↑ LOWER() transforms the column, index is bypassed

-- FIX: create a functional/expression index
CREATE INDEX idx_email_lower ON users(LOWER(email));
-- Or better: store normalized email in a separate column
```

## Anti-Pattern 5: Duplicate & Redundant Indexes

```sql
-- These are redundant (idx_a_b covers queries on just `a`):
CREATE INDEX idx_a ON orders(customer_id);           -- redundant!
CREATE INDEX idx_a_b ON orders(customer_id, status); -- this covers both

-- But NOT the other way around:
CREATE INDEX idx_b ON orders(status);                -- still needed!
CREATE INDEX idx_a_b ON orders(customer_id, status); -- doesn't cover WHERE status = ...
```

Find duplicates with:
```sql
-- PostgreSQL: find redundant indexes
SELECT
  idx1.indexrelid::regclass AS redundant_index,
  idx2.indexrelid::regclass AS covering_index
FROM pg_index idx1
JOIN pg_index idx2 ON idx1.indrelid = idx2.indrelid
  AND idx1.indexrelid != idx2.indexrelid
  AND idx1.indkey <@ idx2.indkey -- idx1 columns are subset of idx2
WHERE idx1.indisunique = false;
```
