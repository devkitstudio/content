## Isolation Levels vs Anomalies

| Anomaly | READ UNCOMMITTED | READ COMMITTED | REPEATABLE READ | SERIALIZABLE |
|---------|------------------|----------------|-----------------|--------------|
| **Dirty Read** | YES | NO | NO | NO |
| **Non-Repeatable Read** | YES | YES | NO | NO |
| **Phantom Read** | YES | YES | YES* | NO |

*PostgreSQL's REPEATABLE READ prevents phantoms in most cases but spec allows them.

## Detailed Anomalies

### Dirty Read: Reading Uncommitted Data

```sql
-- T1 writes, T2 reads before T1 commits
T1: UPDATE account SET balance = 0 WHERE id = 1;
T2: SELECT * FROM account WHERE id = 1;  -- Sees balance = 0
T1: ROLLBACK;
T2: SELECT * FROM account WHERE id = 1;  -- Now sees original balance (inconsistent!)

Protected by: READ COMMITTED and higher
```

### Non-Repeatable Read: Value Changed Mid-Transaction

```sql
-- T1 reads same value twice, gets different results
T1: SELECT balance FROM account WHERE id = 1;  -- $1000
T2: UPDATE account SET balance = 500 WHERE id = 1; COMMIT;
T1: SELECT balance FROM account WHERE id = 1;  -- $500 (different!)

Protected by: REPEATABLE READ and higher
```

### Phantom Read: New Rows Appear

```sql
-- T1 queries range, new rows added before second query
T1: SELECT COUNT(*) FROM orders WHERE created_at > '2026-04-01';  -- 100
T2: INSERT INTO orders (created_at) VALUES ('2026-04-05'); COMMIT;
T1: SELECT COUNT(*) FROM orders WHERE created_at > '2026-04-01';  -- 101 (phantom!)

Protected by: SERIALIZABLE only (or REPEATABLE READ in PostgreSQL)
```

## Performance Impact

```sql
-- Small transaction
Level: READ UNCOMMITTED  | ~0.1ms
Level: READ COMMITTED    | ~0.1ms (default, no overhead)
Level: REPEATABLE READ   | ~0.5ms (snapshots)
Level: SERIALIZABLE      | ~5-10ms (conflict detection)

-- Large transaction (100K rows)
Level: READ UNCOMMITTED  | ~100ms
Level: READ COMMITTED    | ~120ms
Level: REPEATABLE READ   | ~500ms (snapshot overhead)
Level: SERIALIZABLE      | ~5000ms (serialization checks)

-- Highly concurrent (1000 concurrent txns)
Level: READ COMMITTED    | 200ms avg, 1000ms p99
Level: REPEATABLE READ   | 500ms avg, 2000ms p99
Level: SERIALIZABLE      | 5000ms avg, 10000ms p99+ (conflicts/retries)
```

## Locking Behavior

| Level | Row Locks | Intent Locks | Conflicts | Deadlocks |
|-------|-----------|--------------|-----------|-----------|
| READ UNCOMMITTED | No | No | Rare | None |
| READ COMMITTED | Yes (writes) | Yes | Possible | Possible |
| REPEATABLE READ | Yes (writes) | Yes | More likely | Likely |
| SERIALIZABLE | Heavy | Heavy | Very likely | Very likely |

## Testing Isolation Levels

```bash
# Terminal 1: Start transaction
psql -h localhost -U user -d mydb

BEGIN;
SET TRANSACTION ISOLATION LEVEL READ COMMITTED;
SELECT COUNT(*) FROM users;
-- Keep this open, don't commit

# Terminal 2: Modify data
psql -h localhost -U user -d mydb
UPDATE users SET status = 'inactive';

# Back to Terminal 1: Re-query
SELECT COUNT(*) FROM users;
-- Result varies by isolation level!
-- READ COMMITTED: Sees updated data (100ms after UPDATE commits)
-- REPEATABLE READ: Still sees original data
-- SERIALIZABLE: Might error with serialization conflict
```

## Recommended Configuration

```sql
-- OLTP Application (web app)
SET TRANSACTION ISOLATION LEVEL READ COMMITTED;
-- Default, balances speed and safety

-- Batch Processing (ETL, reports)
SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;
-- Stronger consistency for complex calculations

-- Financial Transactions
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;
-- Absolute consistency, retries handled by app

-- Analytics (historical data)
SET TRANSACTION ISOLATION LEVEL READ UNCOMMITTED;  -- If safe
-- Fast, consistency not critical
```

## Migration Checklist

If upgrading from READ COMMITTED to REPEATABLE READ:

- [ ] Test for phantom reads in complex queries
- [ ] Measure performance impact (test both levels)
- [ ] Add retry logic for SERIALIZABLE failures
- [ ] Monitor lock contention and deadlocks
- [ ] Educate team on isolation semantics
- [ ] Document why level was chosen
- [ ] Have rollback plan (revert to READ COMMITTED)

Example retry helper:

```python
def with_isolation_level(level='SERIALIZABLE', max_retries=5):
    def decorator(func):
        def wrapper(*args, **kwargs):
            for attempt in range(max_retries):
                try:
                    cursor.execute(f"SET TRANSACTION ISOLATION LEVEL {level}")
                    cursor.execute("BEGIN")
                    result = func(*args, **kwargs)
                    cursor.execute("COMMIT")
                    return result
                except psycopg2.extensions.TransactionRollbackError:
                    # Serialization failure, retry
                    if attempt < max_retries - 1:
                        time.sleep(0.1 * (2 ** attempt))
                    else:
                        raise
        return wrapper
    return decorator

@with_isolation_level('SERIALIZABLE', max_retries=3)
def critical_operation():
    # This function auto-retries on serialization conflicts
    ...
```
