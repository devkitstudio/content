# The Architectural Flaw of Offset Pagination

To paginate through millions of records efficiently, we must fundamentally shift how the database traverses its indexes.

## The Anti-Pattern: OFFSET / LIMIT

When a database executes an `OFFSET N LIMIT M` query, it cannot magically jump to row `N`. The database engine must scan, fetch, and subsequently **discard** the first `N` rows before returning the next `M` rows.

As the page number increases, the `OFFSET` grows, and the query exhibits **O(N) linear time degradation**. The database spends massive CPU and memory resources reading data it will immediately throw away.

## The Standard: Cursor / Keyset Pagination

Cursor-based pagination (often called Keyset Pagination) eliminates the concept of "pages" and "offsets". Instead of saying _"skip the first 10,000 items"_, it says _"give me 20 items that come immediately after item X"_.

**The B-Tree Index Seek:**
By utilizing a `WHERE` clause on an indexed column (e.g., `WHERE id > last_seen_id`), the database performs an **Index Seek**. It navigates the B-Tree index directly to the exact starting node in **O(1) constant time** and simply reads the next `M` sequential leaf nodes.

Regardless of whether you are fetching the 1st page or the 100,000th page, the execution time remains mathematically flat.

## The Pagination Contract

To obscure the underlying database columns and allow for future architectural changes (like switching from ID to Timestamp), the API must never expose raw database fields as the cursor.

The cursor must be an opaque string (typically Base64 encoded) that the backend decodes to extract the precise database coordinates (the Keyset).
