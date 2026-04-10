## Architectural Trade-offs: Partitioning Strategies

Choosing the wrong partition key or strategy is an irreversible architectural failure that requires complete table reconstruction to fix.

| Strategy  | Mechanism                                 | Optimal Use Case                                              | Operational Advantage                                                          |
| :-------- | :---------------------------------------- | :------------------------------------------------------------ | :----------------------------------------------------------------------------- |
| **Range** | Bounded intervals (Dates, IDs)            | Time-series data, Audit logs, Financial transactions.         | Perfect for sliding-window data retention (dropping data older than X months). |
| **List**  | Discrete exact values (Region, Tenant ID) | Multi-tenant SaaS architectures, Data sovereignty (GDPR).     | Guarantees hard isolation of data clusters.                                    |
| **Hash**  | Modulus algorithm on a key                | High-throughput queues, evenly distributing write contention. | Prevents "hot spots" on the disk when inserts are sequential.                  |

## The Decision Matrix

```text
Do you need to continuously archive or delete old data?
├─ YES → Use RANGE Partitioning (by Date).
└─ NO → Do you need strict tenant/regional isolation?
    ├─ YES → Use LIST Partitioning (by Tenant_ID or Region_Code).
    └─ NO → Is the primary issue Write Contention (hot spots)?
        ├─ YES → Use HASH Partitioning.
        └─ NO → Do not partition. Optimize your indexes and RAM first.
```
