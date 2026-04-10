## Execution: Range Partitioning (PostgreSQL)

**Architectural Mandate:** Do not manually manage partition creation in production. Use established extensions like `pg_partman` or automated triggers to pre-create future partitions to avoid downtime.

### 1. The Logical Definition

Define the parent table. Note that the Primary Key **must** include the partition key.

```sql
-- The Parent Table (Contains no actual data)
CREATE TABLE orders (
    id UUID,
    user_id UUID NOT NULL,
    amount DECIMAL(10,2) NOT NULL,
    created_at TIMESTAMP NOT NULL,
    -- PRIMARY KEY MUST include the partition key
    PRIMARY KEY (id, created_at)
) PARTITION BY RANGE (created_at);
```

### 2. Physical Partition Instantiation

Create the physical tables for specific ranges.

```sql
-- Create partitions (e.g., Monthly)
CREATE TABLE orders_y2024m01 PARTITION OF orders
    FOR VALUES FROM ('2024-01-01') TO ('2024-02-01');

CREATE TABLE orders_y2024m02 PARTITION OF orders
    FOR VALUES FROM ('2024-02-01') TO ('2024-03-01');

-- Indexes are created locally on each physical partition
CREATE INDEX idx_orders_y2024m01_user ON orders_y2024m01(user_id);
CREATE INDEX idx_orders_y2024m02_user ON orders_y2024m02(user_id);
```

### 3. Query Pruning Validation

Always run `EXPLAIN` to ensure the planner is correctly pruning partitions.

```sql
-- Optimal: Query Planner limits the scan to a single partition
EXPLAIN SELECT * FROM orders
WHERE user_id = '123e4567-e89b-12d3-a456-426614174000'
  AND created_at >= '2024-01-15'
  AND created_at < '2024-01-30';
```
