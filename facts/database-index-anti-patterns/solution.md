# Indexing Anti-Patterns

Indexes are not free performance boosters; they are redundant data structures that trade write performance and storage space for read speed.

## Anti-Pattern 1: The "Index Everything" Strategy (Write Amplification)

Every index is a separate B-tree that the database engine must synchronously update during every `INSERT`, `UPDATE`, and `DELETE`.

**The Flaw:** Adding indexes linearly degrades write throughput and bloats memory usage, as index pages compete with raw data pages for space in the RAM (Buffer Pool).
**The Mandate:** Only index columns that are mathematically proven to be bottlenecks in `WHERE`, `JOIN`, or `ORDER BY` clauses of high-frequency queries.

## Anti-Pattern 2: Low-Selectivity Scans

Indexing a column with very few distinct values (e.g., `status ENUM('active', 'inactive')`) when querying for the majority value.

**The Flaw:** The Query Planner evaluates the cost of random IO (Index Scan + Heap Table Fetch) vs. sequential IO (Full Table Scan). If an index returns a large percentage of the table, jumping back and forth between the index and the heap memory is significantly slower than sequentially scanning the entire table. The index becomes dead weight.
**The Fix:** Create **Partial Indexes** if you only care about the minority value (e.g., `CREATE INDEX idx_suspended ON users(status) WHERE status = 'suspended'`).

## Anti-Pattern 3: The Range-First Fallacy (Composite Indexes)

Assuming that the highest cardinality column should always go first in a composite index.

**The Flaw:** B-Trees process composite indexes sequentially. If your query uses a Range operator (`>`, `<`, `BETWEEN`) on the first indexed column, the database **cannot** use the subsequent columns in the index for seeking; it can only use them for filtering.

```sql
-- The Query:
SELECT * FROM orders WHERE created_at > '2026-01-01' AND status = 'paid';

-- FATAL (Range First): CREATE INDEX idx_bad ON orders(created_at, status);
-- Result: Scans ALL dates after Jan 1st, then manually filters 'paid' statuses.

-- OPTIMAL (Equality First): CREATE INDEX idx_good ON orders(status, created_at);
-- Result: Instantly jumps to 'paid' branch, then cleanly scans the date range.
```

**The Mandate:** Rule of thumb for composite indexes: **Equality First, Range Second**.

## Anti-Pattern 4: Blind String Indexing (Bypassing via Functions)

Wrapping an indexed column in a function entirely blinds the database to the B-Tree.

```sql
-- Index: CREATE INDEX idx_email ON users(email);
-- FATAL:
SELECT * FROM users WHERE LOWER(email) = 'john@example.com';
-- The LOWER() function transforms the data, forcing a sequential scan.

-- FIX: Use a Functional Index (Expression Index)
CREATE INDEX idx_email_lower ON users(LOWER(email));
```

## Anti-Pattern 5: Left-Prefix Redundancy

Creating standalone indexes for columns that are already the leading prefix of a composite index.

```sql
CREATE INDEX idx_customer ON orders(customer_id);                 -- REDUNDANT!
CREATE INDEX idx_customer_status ON orders(customer_id, status);  -- COVERS BOTH!
```

Because B-Trees read left-to-right, queries filtering solely on `customer_id` will naturally use the `idx_customer_status` index. The standalone `idx_customer` simply wastes disk space and write cycles.
