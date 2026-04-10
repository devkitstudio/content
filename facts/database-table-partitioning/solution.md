# The Architecture of Table Partitioning

When a database table grows indefinitely, the B-Tree indexes attached to it grow logarithmically. Eventually, the active index size exceeds the database's available RAM (Buffer Pool).

When this happens, the database is forced to swap index pages to disk, causing query latency to degrade exponentially (Cache Eviction). Furthermore, massive tables become virtually impossible to maintain (Vacuuming, Reindexing, or Archiving requires aggressive locks that cause downtime).

## The Partitioning Paradigm

Table Partitioning (Declarative Partitioning) splits a single massive logical table into multiple smaller physical tables (partitions) at the storage engine level.

**The Architectural Mechanisms:**

1.  **Index Shrinkage:** Instead of one massive index, each partition maintains its own isolated index. The "hot" partition's index is small enough to stay entirely in RAM.
2.  **Partition Pruning:** When a query includes the partition key in its `WHERE` clause, the Query Planner instantly discards irrelevant partitions, drastically reducing the dataset scanned.
3.  **O(1) Data Lifecycle:** Deleting historical data transforms from a slow, lock-heavy `DELETE FROM` (which causes massive WAL generation and table bloat) into a near-instant metadata operation (`DROP TABLE partition_name`).
