## The 4 SQL Isolation Levels

### Level 1: READ UNCOMMITTED (Rarely Used)

```sql
SET TRANSACTION ISOLATION LEVEL READ UNCOMMITTED;

-- Transaction A (Write)          Transaction B (Read)
BEGIN;
UPDATE accounts SET balance = 500
WHERE id = 1;
                                  BEGIN;
                                  SELECT SUM(balance) FROM accounts;
                                  -- DIRTY READ: Sees uncommitted 500
ROLLBACK;
                                  SELECT SUM(balance) FROM accounts;
                                  -- Now sees original value (inconsistent!)
                                  COMMIT;
```

**Issues:**
- Dirty reads: See uncommitted changes
- Non-repeatable reads: Value changes mid-transaction
- Phantom reads: New rows appear mid-transaction

**Use:** Almost never. Only for loosely-consistent reporting.

### Level 2: READ COMMITTED (Default, Most Common)

```sql
SET TRANSACTION ISOLATION LEVEL READ COMMITTED;  -- PostgreSQL default

-- Transaction A (Write)          Transaction B (Read)
BEGIN;
SELECT * FROM orders WHERE id = 1;
                                  BEGIN;
                                  UPDATE orders SET status = 'shipped'
                                  WHERE id = 1;
                                  COMMIT;
SELECT * FROM orders WHERE id = 1;
-- Now sees 'shipped' (reads committed data)
COMMIT;
```

**Guarantees:**
- No dirty reads (only sees committed data)
- Non-repeatable reads possible (value changed)
- Phantom reads possible (new rows added)

**Examples of problems:**
```sql
-- Transaction A                  Transaction B
BEGIN;
SELECT SUM(price) FROM orders;    -- Returns $1000
                                  BEGIN;
                                  INSERT INTO orders (price) VALUES (500);
                                  COMMIT;

SELECT SUM(price) FROM orders;
-- Now returns $1500 (phantom read: new row added)
COMMIT;

-- Report: "$1000 total" but database shows "$1500" (inconsistent!)
```

**Use:** Most OLTP systems. Fast, few locks.

### Level 3: REPEATABLE READ

```sql
SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;

-- Transaction A                  Transaction B
BEGIN;
SELECT COUNT(*) FROM orders;
-- Result: 100 rows
                                  BEGIN;
                                  INSERT INTO orders VALUES (...);
                                  INSERT INTO orders VALUES (...);
                                  COMMIT;

SELECT COUNT(*) FROM orders;
-- Still 100 rows (sees snapshot at transaction start)

-- But...
SELECT * FROM orders WHERE id = 999;
-- NEW row is visible (phantom read still possible with INSERT)
COMMIT;
```

**Guarantees:**
- No dirty reads
- No non-repeatable reads (same data seen if you re-query)
- Phantom reads possible (new rows can appear)

**Example problem:**
```sql
-- Inventory system
-- Transaction A                  Transaction B
BEGIN;
SELECT COUNT(*) FROM inventory;
-- Result: 5 items
                                  BEGIN;
                                  INSERT INTO inventory (...);  -- Add item
                                  COMMIT;

-- Check again
SELECT COUNT(*) FROM inventory;
-- Now 6 items (phantom read, new row inserted)

-- Logic breaks: "We had 5 items, expected to remove 3, should have 2 left"
-- But phantom INSERT makes math inconsistent
COMMIT;
```

**Use:** Complex reports, financial calculations. Medium locking.

### Level 4: SERIALIZABLE (Strongest, Most Expensive)

```sql
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;

-- Works as if transactions ran one-at-a-time (serial)
-- Transaction A                  Transaction B
BEGIN;
SELECT COUNT(*) FROM orders;
-- Result: 100
                                  BEGIN;
                                  INSERT INTO orders VALUES (...);
                                  -- Blocks or errors if conflicts with A

SELECT COUNT(*) FROM orders;
-- Still 100 (no changes visible)

-- If B tried to INSERT:
-- ERROR: could not serialize access due to concurrent update
COMMIT;
```

**Guarantees:**
- No dirty reads
- No non-repeatable reads
- No phantom reads (true serialization)

**Use:** Critical operations: financial transactions, inventory, reservations.

## Real-World Examples

### Example 1: Double-Charge Bug (READ COMMITTED)

```sql
-- E-commerce system
-- T1: Check balance                T2: Process payment
BEGIN;
SELECT balance FROM accounts WHERE id = 1;
-- Result: $1000
                                  BEGIN;
                                  UPDATE accounts SET balance = balance - 500
                                  WHERE id = 1;
                                  COMMIT;

-- Check balance again
SELECT balance FROM accounts WHERE id = 1;
-- Result: $500 (non-repeatable read)

-- Process our payment
UPDATE accounts SET balance = balance - 500
WHERE id = 1;
-- New balance: $0 (but should error if < $500)
COMMIT;
```

**Fix: Use SERIALIZABLE**
```sql
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;

BEGIN;
SELECT balance FROM accounts WHERE id = 1 FOR UPDATE;
-- Locks row, T2 waits

if balance >= 500:
    UPDATE accounts SET balance = balance - 500 WHERE id = 1;
COMMIT;
```

### Example 2: Report Inconsistency (REPEATABLE READ)

```sql
-- Finance report counting transactions
-- T1: Generate report           T2: Process new orders
BEGIN;
SELECT COUNT(*), SUM(amount)
FROM orders WHERE status = 'completed';
-- Result: 100 orders, $50000 total
                                  BEGIN;
                                  INSERT INTO orders (...);  -- Add $5000 order
                                  COMMIT;

-- Verify by checking products ordered
SELECT COUNT(DISTINCT product_id) FROM order_items;
-- Result: Different from expected (phantom read: new items)

-- Report generated: "100 orders, $50000"
-- But when checked later: "101 orders, $55000" (inconsistent)
COMMIT;
```

**Fix: Use SERIALIZABLE**
```sql
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;
BEGIN;
SELECT COUNT(*), SUM(amount) FROM orders WHERE status = 'completed';
-- If T2 tries to INSERT, it gets:
-- ERROR: could not serialize access due to concurrent update

-- Report is consistent: 100 orders, $50000
COMMIT;
```

## Choosing the Right Level

| Level | Speed | Consistency | Locking | Use Case |
|-------|-------|-------------|---------|----------|
| READ UNCOMMITTED | Very Fast | Weak | Minimal | Analytics, logs (ok to be slightly stale) |
| READ COMMITTED | Fast | Good | Moderate | OLTP, web apps (default) |
| REPEATABLE READ | Medium | Better | More | Complex queries, some consistency needed |
| SERIALIZABLE | Slow | Perfect | Heavy | Financial, inventory, critical operations |

**Decision Tree:**
```
Is this a critical operation?
├─ YES: Payments, reservations, inventory
│  └─ Use SERIALIZABLE
│
└─ NO: Reads, reporting, caching
   ├─ Do you need strong consistency?
   │  ├─ YES (complex calculations)
   │  └─ Use REPEATABLE READ
   │
   └─ NO (ok to be slightly stale)
      └─ Use READ COMMITTED (default)
```

## PostgreSQL-Specific: Serialization Failures

PostgreSQL uses Serializable Snapshot Isolation (SSI):
- Detects conflicts automatically
- Raises error instead of blocking
- App must retry transaction

```python
# Python example with retry logic
def transfer_money(from_id, to_id, amount, max_retries=3):
    for attempt in range(max_retries):
        try:
            cursor.execute("SET TRANSACTION ISOLATION LEVEL SERIALIZABLE")
            cursor.execute("BEGIN")

            # Check balance
            cursor.execute("SELECT balance FROM accounts WHERE id = %s FOR UPDATE", (from_id,))
            balance = cursor.fetchone()[0]

            if balance < amount:
                raise ValueError("Insufficient funds")

            # Transfer
            cursor.execute("UPDATE accounts SET balance = balance - %s WHERE id = %s", (amount, from_id))
            cursor.execute("UPDATE accounts SET balance = balance + %s WHERE id = %s", (amount, to_id))

            cursor.execute("COMMIT")
            return True

        except psycopg2.Error as e:
            if "serialization failure" in str(e):
                # Retry (exponential backoff)
                import random
                import time
                time.sleep(random.uniform(0.1 * (2 ** attempt), 0.2 * (2 ** attempt)))
                continue
            else:
                raise

    raise RuntimeError(f"Failed after {max_retries} retries")
```
