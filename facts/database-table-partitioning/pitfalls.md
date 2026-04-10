## The Traps of PostgreSQL Partitioning

### 1. The Global Unique Index Limitation

This is the most common reason partitioning fails in legacy migrations.

- **The Flaw:** PostgreSQL does not support Global Unique Indexes across partitions. You cannot enforce a `UNIQUE(id)` constraint on the parent table unless the partition key (`created_at`) is part of the unique constraint.
- **The Consequence:** If your ORM or application logic strictly requires looking up records by a single UUID without knowing the date, you will either have to rewrite the application layer to include the date, or the database will be forced to scan _every single partition_ to find the UUID.

### 2. Query Planner Exhaustion (Too Many Partitions)

Developers often partition too granularly (e.g., Daily partitions for 5 years = 1,825 tables).

- **The Flaw:** The PostgreSQL Query Planner must acquire locks and evaluate the metadata of every single partition during the planning phase. If you have thousands of partitions, the time taken just to _plan_ the query will exceed the time to execute it, bringing the database CPU to 100%.
- **The Fix:** Aim for dozens or low hundreds of partitions, not thousands. Monthly or Weekly partitions are vastly superior to Daily partitions for multi-year retention.

### 3. The `UPDATE` Migration Trap

- **The Flaw:** If you execute an `UPDATE` statement that changes the partition key (e.g., changing `created_at` from January to March), PostgreSQL must physically delete the row from the January partition and insert it into the March partition.
- **The Fix:** Never partition on a column that is mutable. Partition keys must be strictly immutable (Append-Only).
