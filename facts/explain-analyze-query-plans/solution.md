## Reading EXPLAIN ANALYZE Output

Run `EXPLAIN ANALYZE` to see actual execution with timing and row counts:

```sql
EXPLAIN ANALYZE
SELECT id, email, created_at
FROM users
WHERE status = 'active' AND created_at > '2026-01-01';
```

Output structure:
```
Seq Scan on users  (cost=0.00..45000.00 rows=500000 width=32) (actual time=0.045..8234.123 rows=480000)
  Filter: (status = 'active') AND (created_at > '2026-01-01')
  Rows Removed by Filter: 520000
Planning Time: 0.234 ms
Execution Time: 8456.789 ms
```

**Key metrics:**
- `cost=0.00..45000.00` — estimated startup cost..total cost (relative units)
- `rows=500000` — estimated rows returned
- `actual time=0.045..8234.123` — actual startup..total time (ms)
- `actual rows=480000` — actual rows returned
- `Rows Removed by Filter` — filtered out before output

## Node Types & Red Flags

| Node Type | What It Does | Red Flag |
|-----------|------------|----------|
| **Seq Scan** | Reads entire table sequentially | Usually bad on large tables |
| **Index Scan** | Uses index, reads matching rows | Good |
| **Index Only Scan** | Everything is in index, no table access | Optimal |
| **Bitmap Index Scan** | Combines multiple indexes | Good for OR conditions |
| **Hash Join** | Uses hash table (best for large joins) | Check memory |
| **Nested Loop** | Outer loop × inner loop (O(n²)) | Slow on big tables |
| **Merge Join** | Pre-sorted join (good for large sets) | Needs both inputs sorted |

## Common EXPLAIN Pitfalls

**Estimated rows ≠ actual rows** (wildly off):
- Statistics are stale → run `ANALYZE`
- Column distribution unknown → check `n_distinct`

**High "Rows Removed by Filter"**:
- Filter not pushed to index
- Need index on filter column

**Nested Loop with large inner table**:
- Need hash join
- Add index to inner table's join key

## Quick Optimization Checklist

1. **Is there a sequential scan on a large table?**
   ```sql
   -- Check if index exists
   SELECT * FROM pg_indexes WHERE tablename = 'users';
   ```

2. **Are filter columns indexed?**
   ```sql
   -- Add missing index
   CREATE INDEX idx_users_status_created
   ON users(status, created_at);
   ```

3. **Is estimated != actual?**
   ```sql
   ANALYZE users;
   ```

4. **Any nested loops joining large tables?**
   ```sql
   -- Join on indexed foreign key
   SELECT * FROM orders o
   JOIN customers c ON o.customer_id = c.id  -- needs index on c.id
   WHERE c.status = 'active';
   ```

5. **Check buffer cache hits:**
   ```sql
   EXPLAIN (ANALYZE, BUFFERS)
   SELECT * FROM users WHERE id = 42;
   -- Buffers: shared hit=3 read=1 means mostly in RAM
   ```
