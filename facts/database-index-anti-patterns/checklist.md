## The Execution Checklist: Adding an Index

Do not guess. Adding an index must be an evidence-based operation.

### Before Adding (The Justification Phase)

1.  **Prove the Bottleneck:** Run `EXPLAIN ANALYZE` on the target query. Confirm it is executing a `Seq Scan` (Sequential Scan) and consuming significant relative execution time.
2.  **Evaluate Selectivity:** Will this index filter out the vast majority of rows? If the query matches a massive chunk of the table, an index will likely be ignored by the planner.
3.  **Check Prefix Coverage:** Review `\d table_name` (Postgres) or `SHOW INDEXES` (MySQL). Ensure a composite index doesn't already exist where your target column is the leftmost prefix.
4.  **Assess the Write/Read Ratio:** Is this an immutable log table (append-only, high throughput) or a read-heavy configuration table? High-ingestion tables should have absolute minimal indexing.

### After Adding (The Validation Phase)

1.  **Verify Usage:** Run `EXPLAIN ANALYZE` again. The execution plan **must** shift from a `Seq Scan` to an `Index Scan` or `Index Only Scan`. If it doesn't, drop the index immediately.
2.  **Monitor Dead Weight:** Unused indexes are pure technical debt. Periodically audit your database for indexes that have never been scanned since the last statistics reset.

```sql
-- PostgreSQL: Detect Dead Weight Indexes
SELECT
  schemaname, tablename, indexrelname,
  idx_scan, -- 0 indicates the index has never been used for a read
  pg_size_pretty(pg_relation_size(indexrelid)) AS size
FROM pg_stat_user_indexes
WHERE idx_scan = 0
ORDER BY pg_relation_size(indexrelid) DESC;
```

### The Architectural Decision Matrix

| Scenario                                                  | Index Requirement      | Strategy                                                                 |
| :-------------------------------------------------------- | :--------------------- | :----------------------------------------------------------------------- |
| High-Selectivity `WHERE` clause (UUID, Email)             | **Mandatory**          | Standard B-Tree.                                                         |
| Foreign Key (`JOIN` conditions)                           | **Mandatory**          | Standard B-Tree (prevents full table locks during cascades).             |
| Exact match on Minority state (e.g., `status = 'failed'`) | **Highly Recommended** | Partial Index (`WHERE status = 'failed'`).                               |
| Sorting (`ORDER BY`) with Pagination (`LIMIT`)            | **Highly Recommended** | Composite Index matching the exact sort direction (`ASC`/`DESC`).        |
| High-Ingestion / Append-Only Tables                       | **Avoid**              | Rely on Partitioning instead of deep B-Trees.                            |
| Leading Wildcard (`LIKE '%keyword'`)                      | **Useless**            | B-Trees read left-to-right. Use Full-Text Search (GIN) or Elasticsearch. |
