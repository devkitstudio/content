The traditional way developers implement pagination is by using the `OFFSET` clause.

```sql
SELECT * FROM users ORDER BY created_at DESC LIMIT 20 OFFSET 1000000;
```

While it seems simple, the mechanical reality of how relational databases process the `OFFSET` command is disastrous at scale. 

### The Mechanical Failure
A relational database like PostgreSQL or MySQL does not physically store rows in an ordered array where it can simply jump to index `1,000,000`. 

When executing an `OFFSET 1000000 LIMIT 20`, the database engine must:
1. Load the corresponding index.
2. Sequentially count out **1,000,020** records that match the sorting criteria.
3. Load the data into the memory buffer.
4. Iterate from row `1` to row `1,000,000`, internally throwing each row away (a waste of calculation).
5. Finally grab the last 20 rows and return them over the network.

### Linear Degradation
Because the database is forced to count and discard records, the query time grows linearly. Page 2 might take `2ms`, but Page 50,000 might take `8 seconds`. 

If multiple users attempt to navigate to deep pages simultaneously, the database is forced to load millions of discarded rows into memory concurrently. This triggers severe Disk I/O bottlenecks and inevitably exhausts the connection pool, causing a complete system outage.
