### 1. The UX Illusion
Many engineers obsess over optimizing SQL queries to allow users to navigate to "Page 100,000" of a 2-million record dataset. In reality, **this is a Product UX failure.**

No sane user manually clicks through 100,000 pages to find a specific result. If a user hasn't found what they are looking for within the first 10 pages, they will inevitably adjust their search filters. 

**The Hard Capping Solution:** 
Instead of fighting database constraints, apply a UX "Hard Limit". Exactly like Google Search, only calculate and allow navigation for the first 50 to 100 pages. If a user reaches the cap, display a prompt: *"Please refine your search filters to narrow down the results."* This instantly eliminates the need for deep database offsets purely through UX manipulation.

---

### 2. The `SELECT COUNT(*)` Outage
Offset pagination interfaces traditionally require displaying the total number of pages (e.g., `Page 1 of 54,000`). To calculate the total pages, developers often execute a full table count alongside the data query:

```sql
SELECT COUNT(*) FROM users WHERE status = 'active';
```

**The Incident:** In a prominent production outage, an E-commerce platform attempted to render a massive category page. While the data query (`LIMIT 20`) executed quickly, the accompanying `COUNT(*)` query on an unindexed, heavily-filtered 20-million row table triggered a full sequential scan. 

The query locked the database thread for 45 seconds. As thousands of users clicked the category, the connection pool was instantly exhausted by overlapping `COUNT(*)` queries, resulting in a total blackout of the backend API.

**The Fix:** 
- Never execute live count queries on massive datasets.
- Use query estimators: `EXPLAIN SELECT...` in PostgreSQL to grab an approximate row count from metadata.
- Or, maintain a separate asynchronous Counter Table / Redis cache specifically for tracking the total row volume.
