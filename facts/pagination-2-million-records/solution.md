The industry-standard solution for paginating high-volume data is **Cursor-Based Pagination** (also known as Keyset Pagination).

Instead of relying on the database to count offsets, cursor pagination relies on providing a physical reference point (the "cursor") from the last retrieved dataset. The database can then look up this reference point directly using a B-Tree index.

### The Implementation
When fetching the first page, you query normally:
```sql
SELECT id, title, created_at 
FROM articles 
ORDER BY id DESC 
LIMIT 20;
```

The application extracts the `id` of the last item in the response (e.g., `id: 85293`). When the user scrolls down or requests the next page, the client sends this `id` back as the cursor:

```sql
SELECT id, title, created_at 
FROM articles 
WHERE id < 85293 
ORDER BY id DESC 
LIMIT 20;
```

### Why it excels at scale
1. **`O(1)` Query Stability:** Because the `WHERE id < 85293` clause leverages an Index Seek, the database jumps directly to the cursor's location in the B-Tree. It doesn't matter if you are on page 2 or page 50,000—the query speed remains consistently instant.
2. **Data Consistency:** Unlike Offset pagination, Cursor pagination is immune to data drift. If new records are inserted at the top of the table while a user is paginating, the user will not see duplicate records pushed onto their current page.

### The Trade-off
Cursor-based pagination enforces a strictly sequential navigational flow. It is perfect for **Infinite Scroll** interfaces (like social media feeds) and "Next/Previous" buttons, but it completely drops support for "Jump to Page X" functionality.
