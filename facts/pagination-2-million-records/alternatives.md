When dealing with millions of records, Cursor pagination is the gold standard. However, varying product requirements often demand alternative architectural strategies.

### 1. Deferred Joins
**Use Case:** CMS Admin dashboards where users strictly require a traditional UI with explicit pagination numbers (Page 1, 2... 100) allowing them to jump back and forth.

If traditional OFFSET is too slow, and Cursor pagination cannot jump pages, **Deferred Joins** bridge the gap. 

A Deferred Join forces the database to evaluate the `OFFSET` purely on an Index rather than scanning the heavy primary data table. The database scans the tiny, memory-resident index to find the targeted 20 IDs, and then `JOIN`s back to the primary table to fetch the full row payloads only for those 20 IDs.

```sql
SELECT * FROM users 
INNER JOIN (
    SELECT id FROM users 
    ORDER BY created_at DESC 
    LIMIT 20 OFFSET 500000
) AS deferred_page USING(id);
```
While not `O(1)` like a Cursor, this dramatically reduces the overall Disk I/O penalties associated with deep offsets.

### 2. Search Engines (Elasticsearch)
**Use Case:** E-commerce catalogs where the 2-million record pagination is heavily polluted by multi-dimensional filtering requirements (Price Range, Colors, Categories, Geo-spatial sorting).

Relational databases fall apart when trying to construct B-Tree indexes for every possible permutation of dynamic filters. When complex search queries meet deep pagination, shifting the workload is required.

- **Architecture:** Utilize Change Data Capture (CDC) like Debezium to replicate database rows into a dedicated Search Engine cluster (like **Elasticsearch**).
- **Execution:** Elasticsearch is explicitly built for distributed inverted-indexing. You can utilize its native `search_after` API to flawlessly paginate through heavily fractured datasets in milliseconds.
