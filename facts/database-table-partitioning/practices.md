## Operational Best Practices & Drills

Designing a partitioned table is only 20% of the work. The remaining 80% is operationalizing it so the database survives long-term data growth and automated maintenance cycles.

Below is the standard operating procedure, cemented through a real-world incident drill.

---

### The Core Mandates

1.  **Strict Naming Conventions:** Use the format `[table_name]_y[YYYY]m[MM]` (e.g., `orders_y2024m01`). Never use abbreviations or custom suffixes. Alphabetical sorting must natively match chronological sorting.
2.  **The "N+2" Pre-Creation Rule:** Always maintain an active buffer of `N + 2` future partitions. Never rely on a script to create a partition on the exact day it is needed.
3.  **The Circuit Breaker:** Enforce a `statement_timeout` on the application role. Queries missing the partition key will attempt a catastrophic Full Partition Scan. Kill them before they kill the database.

---

### 🛑 Live Drill: The End-of-Month Crisis

**The Scenario:** It is 11:00 PM on February 28th. PagerDuty alerts you that the automated partition creation script has been silently failing for a week. Tomorrow is March 1st. If the clock strikes midnight and the March partition does not exist, all incoming `INSERT` statements will crash with a `partition key routing error`. A global outage is imminent.

Furthermore, historical partitions from exactly 6 months ago (August of the previous year) are still fully active, bloating the auto-vacuum process.

**The Mission:**

1.  **Emergency Rescue (N+2):** Immediately provision partitions for March and April to restore the safety buffer.
2.  **Cold Storage (Maintenance):** Perform a hard freeze and compression on the August partition to free up RAM/Buffer Pool resources.

#### The Execution

Run the following SQL block directly against the primary instance (using a DBA role with DDL privileges).

```sql
-- MISSION 1: EMERGENCY RESCUE (N+2 PROVISIONING)
BEGIN;

-- 1.1 Create March Partition (Tomorrow)
CREATE TABLE transactions_y2026m03 PARTITION OF transactions
    FOR VALUES FROM ('2026-03-01') TO ('2026-04-01');

CREATE INDEX CONCURRENTLY idx_trans_y2026m03_account
    ON transactions_y2026m03(account_id);

-- 1.2 Create April Partition (Restoring the N+2 Buffer)
CREATE TABLE transactions_y2026m04 PARTITION OF transactions
    FOR VALUES FROM ('2026-04-01') TO ('2026-05-01');

CREATE INDEX CONCURRENTLY idx_trans_y2026m04_account
    ON transactions_y2026m04(account_id);

COMMIT;


-- MISSION 2: COLD STORAGE COMPRESSION
-- Target: August 2025 (transactions_y2025m08)

-- 2.1 Disable Auto-vacuum on the historical partition
ALTER TABLE transactions_y2025m08 SET (
    autovacuum_enabled = false,
    toast.autovacuum_enabled = false
);

-- 2.2 Reclaim dead tuples and physically compress the file
-- WARNING: Requires Access Exclusive Lock. Only run during off-peak hours!
VACUUM FULL transactions_y2025m08;

-- 2.3 Rebuild the index cleanly to eliminate bloat
REINDEX TABLE transactions_y2025m08;
```

---

### 🔬 The Autopsy (Architectural Notes)

Why is the script written this way? Below are the critical operational notes that every DBA must understand:

- **Why wrap Mission 1 in a `BEGIN...COMMIT` block?**
  Creating multiple partitions involves executing DDL (Data Definition Language) statements. In PostgreSQL, DDL is transactional. If the creation of the April partition fails, the March partition creation is rolled back. This prevents the database schema from entering an inconsistent, partial state.
- **The Life-Saving `CONCURRENTLY` Keyword:**
  When building indexes on new or active partitions, you **must** use `CREATE INDEX CONCURRENTLY`. A standard `CREATE INDEX` acquires a lock that blocks all writes to the table. `CONCURRENTLY` allows the database to build the index in the background while continuing to serve production traffic.
- **Why disable `autovacuum` in Mission 2?**
  Data from August of the previous year is considered Historical/Immutable. Customers will never `UPDATE` or `DELETE` past transactions. Allowing the Auto-vacuum daemon to continuously scan a table that never changes is a massive waste of disk I/O. Freeze it.
- **The Red Alert on `VACUUM FULL`:**
  `VACUUM FULL` creates a completely new physical copy of the table, compacting it to the absolute minimum size. However, it requires an **Access Exclusive Lock** (blocking all reads and writes to that specific partition). This is why it is strictly reserved for old partitions and must be executed during low-traffic windows.
