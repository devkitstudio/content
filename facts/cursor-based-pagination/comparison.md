# Pagination Methods Comparison

## Performance Benchmark (1M Products)

| Method | Page 1 | Page 100 | Page 10k | Page 100k | Memory |
|--------|--------|----------|----------|-----------|--------|
| OFFSET/LIMIT | 5ms | 15ms | 300ms | 5000ms | High |
| ID Cursor | 3ms | 3ms | 3ms | 3ms | Low |
| Timestamp Cursor | 4ms | 4ms | 4ms | 4ms | Low |
| Keyset | 2ms | 2ms | 2ms | 2ms | Low |

## Detailed Comparison

### OFFSET/LIMIT (Traditional)
```javascript
// SELECT * FROM users OFFSET 1000 LIMIT 20
// Scans 1020 rows, returns 20

app.get('/api/users', (req, res) => {
  const page = req.query.page || 1;
  const limit = 20;
  const offset = (page - 1) * limit;

  const users = await db.query(
    'SELECT * FROM users LIMIT ? OFFSET ?',
    [limit, offset]
  );

  res.json({
    items: users,
    page,
    hasNextPage: users.length === limit
  });
});
```

**Pros:**
- Simple to understand
- Easy to calculate total pages
- Can jump to any page

**Cons:**
- Slow at high offsets
- Performance degrades as data grows
- Can't handle concurrent insertions well (data shifts)

**Use when:**
- Dataset is small (< 100k rows)
- Users need random page access
- You can afford the performance cost

---

### ID Cursor (Keyset Pagination)
```javascript
// SELECT * FROM users WHERE id > 5000 LIMIT 20
// No scanning, uses index seek

app.get('/api/users', (req, res) => {
  const limit = 20;
  const cursor = req.query.cursor; // Last ID from previous page

  let query = 'SELECT * FROM users WHERE 1=1';
  const params = [];

  if (cursor) {
    query += ' AND id > ?';
    params.push(cursor);
  }

  query += ' ORDER BY id ASC LIMIT ?';
  params.push(limit + 1);

  const users = await db.query(query, params);
  const hasMore = users.length > limit;
  const items = users.slice(0, limit);

  res.json({
    items,
    nextCursor: hasMore ? items[items.length - 1].id : null
  });
});
```

**Pros:**
- Constant time complexity O(1)
- Fast at any position
- Handles concurrent inserts well
- Low memory usage

**Cons:**
- Can't jump to arbitrary page
- Can't calculate total pages easily
- Must always move forward

**Use when:**
- Performance is critical
- Large datasets (> 100k rows)
- Building feeds or search results
- Mobile apps with limited bandwidth

---

### Timestamp Cursor (For Feeds)
```javascript
// SELECT * FROM posts WHERE created_at < ? LIMIT 20
// Useful for feeds where new items arrive

app.get('/api/feed', (req, res) => {
  const limit = 20;
  const cursor = req.query.cursor; // ISO timestamp

  let query = 'SELECT * FROM posts WHERE 1=1';
  const params = [];

  if (cursor) {
    query += ' AND created_at < ?';
    params.push(new Date(cursor));
  }

  query += ' ORDER BY created_at DESC LIMIT ?';
  params.push(limit + 1);

  const posts = await db.query(query, params);
  const hasMore = posts.length > limit;
  const items = posts.slice(0, limit);

  res.json({
    items,
    nextCursor: hasMore ? items[items.length - 1].created_at : null
  });
});
```

**Pros:**
- Natural for time-based data
- Handles new items arriving
- Intuitive for users (chronological)

**Cons:**
- Duplicates possible if multiple items have same timestamp
- Less stable if records updated frequently

**Use when:**
- Displaying feeds, timelines
- Data is time-sorted
- New items arrive constantly

---

### Keyset/Seek Pagination (Advanced)
```javascript
// Combine multiple fields for stable pagination
// SELECT * FROM products WHERE (price, id) > (99.99, 5000) LIMIT 20

app.get('/api/products', (req, res) => {
  const limit = 20;
  const cursor = req.query.cursor; // { price: 99.99, id: 5000 }

  let query = 'SELECT * FROM products WHERE 1=1';
  const params = [];

  if (cursor) {
    // Cursor pagination with composite key
    query += ` AND (price, id) > (?, ?)`;
    params.push(cursor.price, cursor.id);
  }

  query += ' ORDER BY price ASC, id ASC LIMIT ?';
  params.push(limit + 1);

  const products = await db.query(query, params);
  const hasMore = products.length > limit;
  const items = products.slice(0, limit);

  const nextCursor = hasMore
    ? { price: items[items.length - 1].price, id: items[items.length - 1].id }
    : null;

  res.json({
    items,
    nextCursor: nextCursor ? Buffer.from(JSON.stringify(nextCursor)).toString('base64') : null
  });
});
```

**Pros:**
- Stable pagination regardless of insertion/deletion
- Handles complex sort criteria
- O(1) performance

**Cons:**
- More complex implementation
- Requires composite indexes
- Cursor is less meaningful to users

**Use when:**
- Complex sorting requirements
- Stability is important
- Performance is critical

---

## When to Use Each

```javascript
// E-commerce product catalog
app.get('/api/products', cursor_pagination); // Fast, no need to jump

// Blog posts with pagination controls
app.get('/api/posts', offset_pagination); // Users want to jump to page 5

// Social media feed
app.get('/api/feed', timestamp_cursor); // New content arrives, chronological

// Search results with sorting
app.get('/api/search', keyset_pagination); // Sorted, stable, efficient
```

## Migration Strategy

```javascript
// Phase 1: Support both methods
app.get('/api/items', (req, res) => {
  if (req.query.page) {
    // Old: offset/limit
    return offsetPagination(req, res);
  } else {
    // New: cursor-based
    return cursorPagination(req, res);
  }
});

// Phase 2: Deprecate offset/limit
// Add warning headers when offset is used

// Phase 3: Remove offset/limit
// Only support cursor pagination
```
