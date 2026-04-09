## Execution: Cursor-Based Pagination Implementation

**Architectural Mandate:** The cursor must be opaque to the client. The client should treat it as a random string token.

### 1. Database Indexing (Prerequisite)

Cursor pagination mathematically fails without a proper B-Tree index matching the exact sort order of your query.

```sql
-- Single-field cursor (ID)
CREATE INDEX idx_products_id ON products(id);

-- Multi-field keyset cursor (e.g., sorted by price, then ID as a tie-breaker)
CREATE INDEX idx_products_price_id ON products(price ASC, id ASC);
```

### 2. The Application Logic (Node.js Example)

A robust implementation for complex sorting (Keyset Pagination) requires decoding the cursor and applying a composite `WHERE` clause.

```javascript
// Utility: Opaque Cursor Encoding/Decoding
const encodeCursor = (payload) =>
  Buffer.from(JSON.stringify(payload)).toString("base64");
const decodeCursor = (token) =>
  JSON.parse(Buffer.from(token, "base64").toString());

// Route Handler
app.get("/api/products", async (req, res) => {
  // Configurable bounds, do not hardcode limit sizes in SQL
  const maxLimit = parseInt(process.env.PAGINATION_MAX_LIMIT) || 100;
  const limit = Math.min(parseInt(req.query.limit) || 20, maxLimit);

  const cursorToken = req.query.cursor;

  let query = "SELECT * FROM products WHERE 1=1";
  const params = [];

  if (cursorToken) {
    const cursor = decodeCursor(cursorToken);
    // Composite Keyset Logic: Fetch items where price is greater,
    // OR where price is equal but ID is greater (tie-breaker)
    query += ` AND (price > ? OR (price = ? AND id > ?))`;
    params.push(cursor.price, cursor.price, cursor.id);
  }

  // Mandatory tie-breaker (id) guarantees deterministic ordering
  query += " ORDER BY price ASC, id ASC LIMIT ?";

  // Fetch limit + 1 to determine if a "next" page exists without a COUNT() query
  params.push(limit + 1);

  const products = await db.query(query, params);

  const hasMore = products.length > limit;
  const items = products.slice(0, limit); // Remove the extra look-ahead item

  // Generate the next opaque token based on the last item's composite keys
  const nextCursor = hasMore
    ? encodeCursor({
        price: items[items.length - 1].price,
        id: items[items.length - 1].id,
      })
    : null;

  res.json({
    data: items,
    meta: { next_cursor: nextCursor, has_more: hasMore },
  });
});
```
