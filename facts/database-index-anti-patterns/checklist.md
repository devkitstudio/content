## Before Adding an Index

1. **Run `EXPLAIN ANALYZE`** on the slow query first. Confirm it's doing a Seq Scan.
2. **Check cardinality**: `SELECT COUNT(DISTINCT col) / COUNT(*) FROM table`. If < 0.1 (less than 10% unique), the index probably won't help.
3. **Check existing indexes**: make sure you're not creating a duplicate or redundant index.
4. **Consider the write cost**: high-write tables (logs, events) should have minimal indexes.

## After Adding an Index

1. Run `EXPLAIN ANALYZE` again. Confirm the planner actually uses the new index.
2. Monitor `pg_stat_user_indexes` (PostgreSQL) for unused indexes:

```sql
-- Find indexes that have NEVER been used since last stats reset
SELECT
  schemaname, tablename, indexrelname,
  idx_scan,          -- 0 = never used
  pg_size_pretty(pg_relation_size(indexrelid)) AS size
FROM pg_stat_user_indexes
WHERE idx_scan = 0
ORDER BY pg_relation_size(indexrelid) DESC;
```

3. Drop unused indexes after confirming they're not needed for constraints.

## Quick Decision Matrix

| Scenario | Index? |
|----------|--------|
| `WHERE` on high-cardinality column (email, UUID) | Yes |
| `WHERE` on low-cardinality column (status, boolean) | Usually no |
| `JOIN` foreign keys | Yes (always index FK columns) |
| `ORDER BY` + `LIMIT` on large tables | Yes (covering index) |
| Write-heavy table (100K+ inserts/day) | Minimal indexes only |
| Small table (< 10K rows) | Probably not worth it |
| `LIKE '%keyword%'` (leading wildcard) | No (use full-text search) |
| `LIKE 'prefix%'` (trailing wildcard) | Yes (B-tree handles this) |
