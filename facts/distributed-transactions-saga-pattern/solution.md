You cannot strictly "rollback" a commit that has already happened on another database. The Senior Architecture standard to solve this is the **Saga Pattern**.

### The Concept
Instead of a single giant transaction, a Saga breaks the workflow down into a sequence of isolated, local transactions. If one local transaction fails, the Saga executes **Compensating Transactions** that undo the changes made by the preceding local transactions.

### The Saga Flow
Imagine a 3-step E-Commerce checkout:
1. **Order Service:** Creates Order `PENDING`.
2. **Inventory Service:** Deducts stock amount (SUCCESS).
3. **Payment Service:** Charges credit card... (FAILS! Card declined).

**The Reversal (Compensation):**
Because Step 3 failed:
- The Saga triggers a *Compensating Transaction* to the **Inventory Service**: `Add stock amount back`.
- The Saga triggers a *Compensating Transaction* to the **Order Service**: `Update Order status to FAILED`.
