If your business logic is highly complex and an Atomic Update isn't enough (e.g. you need to check user limits, VIP status, and stock simultaneously), use **Optimistic Locking**.

### The Concept
Add a `version` (or `updated_at` timestamp) column to your table. You assume conflicts are rare, so you don't lock anything. You just verify the version hasn't changed before you commit.

### The Flow
1. User A reads: `stock: 1`, `version: 1`
2. User B reads: `stock: 1`, `version: 1`
3. User A is ready to buy. They run an update *asserting* the version is still 1:
   ```sql
   UPDATE products SET stock = 0, version = 2 WHERE id = 99 AND version = 1;
   ```
   **Success!** (Rows affected: 1. Stock becomes 0, Version becomes 2).
4. A millisecond later, User B tries to buy using what they read:
   ```sql
   UPDATE products SET stock = 0, version = 2 WHERE id = 99 AND version = 1;
   ```
   **Failed!** (Rows affected: 0. Because version is now 2!). User B's backend detects 0 rows affected and cancels the transaction.
