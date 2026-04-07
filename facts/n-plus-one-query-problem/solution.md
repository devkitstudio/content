Never query the database inside a loop. You must use **Eager Loading** (fetching related data ahead of time) or a **Data Loader** pattern.

### 1. SQL JOIN Approach
Combine the tables at the SQL level so the database does the heavy lifting in a single trip.
```sql
SELECT articles.*, users.name as author_name 
FROM articles 
JOIN users ON articles.author_id = users.id 
LIMIT 100;
```
**Total Queries: 1.**

### 2. The "IN" Clause Approach
Filter related data using a batch of IDs.
```sql
-- Query 1
SELECT * FROM articles LIMIT 100;

-- Extract the 100 unique author IDs in your code, then Query 2
SELECT * FROM users WHERE id IN (1, 2, 8, 14...); 
```
**Total Queries: 2.** (Massive improvement over 101!)
