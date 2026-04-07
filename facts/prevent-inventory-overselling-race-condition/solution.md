The absolute simplest and most performant way to prevent overselling is to **delegate the condition check to the Database engine itself** using an Atomic Update.

Instead of reading the stock into your Node.js/PHP memory, evaluating it, and then saving it, you force the Database to evaluate it *at the exact moment of the write*.

```sql
UPDATE products 
SET stock = stock - 1 
WHERE id = 99 AND stock > 0;
```

### Why it works:
Because standard relational databases (Postgres, MySQL) execute row updates sequentially (atomically).
1. The database receives User A and User B's queries simultaneously.
2. It locks the row for User A, decrements `stock` from 1 to 0. It returns `1 row affected`. (User A successfully buys).
3. It immediately processes User B's query. But now `stock` is 0, so the `WHERE stock > 0` condition fails! It returns `0 rows affected`.
4. Your backend code checks if `rowsAffected === 0` and throws an "Out of stock" error to User B.
