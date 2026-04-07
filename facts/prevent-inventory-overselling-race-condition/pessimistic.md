When money relies on strict consistency and you cannot afford race conditions anywhere in the workflow, use **Pessimistic Locking**.

### The Concept
You assume a conflict *will* happen. The absolute first thing you do is politely tell the database: *"Lock this row. Nobody else is allowed to read or write to this row until I am finished."*

### The Flow
You open a Database Transaction and use `SELECT ... FOR UPDATE`.

```sql
BEGIN TRANSACTION;

-- 1. This locks the row! 
-- If User B tries to run the same query, User B's request will hang and WAIT here.
SELECT stock FROM products WHERE id = 99 FOR UPDATE;

-- 2. Your backend now has exclusive access to evaluate conditions safely
-- (stock > 0) ? continue : rollback

-- 3. Update the stock
UPDATE products SET stock = 0 WHERE id = 99;

COMMIT; 
-- 4. The lock is released. 
-- User B's query un-freezes, reads the new stock (0), and fails safely!
```

> [!CAUTION]  
> Pessimistic locking creates sequential bottlenecks. If 1,000 users try to buy the product, 999 of them are stuck Waiting in a queue on the database level. This can exhaust Database Connection Pools and crash your server under extreme Flash Sale loads. Use Atomic Updates or Redis queues for high-traffic environments.
