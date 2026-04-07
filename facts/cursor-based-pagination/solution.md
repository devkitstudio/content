# Cursor-Based Pagination Implementation

## The Problem: OFFSET/LIMIT Performance Degrades

```javascript
// SLOW - O(n) complexity
// SELECT * FROM products OFFSET 1000000 LIMIT 20
// Scans 1M rows, then takes first 20

app.get('/api/products', async (req, res) => {
  const page = req.query.page || 1;
  const limit = req.query.limit || 20;
  const offset = (page - 1) * limit;

  const products = await db.query(
    'SELECT * FROM products ORDER BY id ASC LIMIT ? OFFSET ?',
    [limit, offset]
  );

  res.json(products);
});

// Response time vs page number
// Page 1: 10ms
// Page 100: 50ms
// Page 1000: 500ms
// Page 10000: 5000ms (!)
```

## The Solution: Cursor-Based Pagination

### Option 1: ID-Based Cursors (Simplest)

```javascript
// FAST - O(1) complexity
// SELECT * FROM products WHERE id > 12345 ORDER BY id ASC LIMIT 20

app.get('/api/products', async (req, res) => {
  const limit = Math.min(parseInt(req.query.limit) || 20, 100);
  const cursor = req.query.cursor; // ID after which to start

  let query = 'SELECT * FROM products WHERE 1=1';
  const params = [];

  if (cursor) {
    query += ' AND id > ?';
    params.push(parseInt(cursor));
  }

  query += ' ORDER BY id ASC LIMIT ?';
  params.push(limit + 1); // Fetch one extra to know if more exist

  const products = await db.query(query, params);

  // Check if there are more results
  const hasMore = products.length > limit;
  const items = products.slice(0, limit);

  // Last item's ID becomes the next cursor
  const nextCursor = hasMore ? items[items.length - 1].id : null;

  res.json({
    items,
    pagination: {
      nextCursor,
      hasMore
    }
  });
});

// Client usage:
// GET /api/products
// GET /api/products?cursor=12345
// GET /api/products?cursor=67890
// Response: { items: [...], pagination: { nextCursor: 99999, hasMore: true } }
```

### Option 2: Timestamp-Based Cursors (Better for Feeds)

```javascript
// For feeds where new items arrive constantly
app.get('/api/feed', async (req, res) => {
  const limit = Math.min(parseInt(req.query.limit) || 20, 100);
  const cursor = req.query.cursor; // ISO timestamp like "2024-12-15T10:30:00Z"

  let query = 'SELECT * FROM posts';
  const params = [];

  if (cursor) {
    // For reverse chronological (newest first)
    query += ' WHERE created_at < ?';
    params.push(new Date(cursor));
  } else {
    // Default to "now"
    params.push(new Date());
  }

  query += ' ORDER BY created_at DESC, id DESC LIMIT ?';
  params.push(limit + 1);

  const posts = await db.query(query, params);

  const hasMore = posts.length > limit;
  const items = posts.slice(0, limit);
  const nextCursor = hasMore ? items[items.length - 1].created_at.toISOString() : null;

  res.json({
    items,
    pagination: {
      nextCursor,
      hasMore
    }
  });
});

// For APIs, encode cursor as base64 to obscure implementation
function encodeCursor(data) {
  return Buffer.from(JSON.stringify(data)).toString('base64');
}

function decodeCursor(cursor) {
  return JSON.parse(Buffer.from(cursor, 'base64').toString());
}

const nextCursor = hasMore ? encodeCursor({ id: items[items.length - 1].id }) : null;
```

### Option 3: Complex Multi-Field Cursors

```javascript
// When you need to sort by multiple fields
app.get('/api/products', async (req, res) => {
  const limit = parseInt(req.query.limit) || 20;
  const cursor = req.query.cursor ? decodeCursor(req.query.cursor) : null;

  let query = 'SELECT * FROM products WHERE 1=1';
  const params = [];

  if (cursor) {
    // Cursor contains: { price: 99.99, id: 12345 }
    // Get items where price < 99.99 OR (price = 99.99 AND id > 12345)
    query += ` AND (
      price < ? OR
      (price = ? AND id > ?)
    )`;
    params.push(cursor.price, cursor.price, cursor.id);
  }

  query += ' ORDER BY price ASC, id ASC LIMIT ?';
  params.push(limit + 1);

  const products = await db.query(query, params);

  const hasMore = products.length > limit;
  const items = products.slice(0, limit);

  const nextCursor = hasMore
    ? encodeCursor({ price: items[items.length - 1].price, id: items[items.length - 1].id })
    : null;

  res.json({
    items,
    pagination: { nextCursor, hasMore }
  });
});

function decodeCursor(cursor) {
  return JSON.parse(Buffer.from(cursor, 'base64').toString());
}

function encodeCursor(data) {
  return Buffer.from(JSON.stringify(data)).toString('base64');
}
```

## Backward Compatibility with OFFSET/LIMIT

```javascript
// Support both pagination styles during migration
app.get('/api/products', async (req, res) => {
  const useOffset = req.query.page !== undefined;

  if (useOffset) {
    // Legacy: offset/limit pagination
    const page = parseInt(req.query.page) || 1;
    const limit = Math.min(parseInt(req.query.limit) || 20, 100);
    const offset = (page - 1) * limit;

    const total = await db.query('SELECT COUNT(*) as count FROM products');
    const products = await db.query(
      'SELECT * FROM products ORDER BY id ASC LIMIT ? OFFSET ?',
      [limit, offset]
    );

    return res.json({
      items: products,
      pagination: {
        page,
        limit,
        total: total[0].count,
        totalPages: Math.ceil(total[0].count / limit)
      }
    });
  }

  // Modern: cursor pagination
  const limit = Math.min(parseInt(req.query.limit) || 20, 100);
  const cursor = req.query.cursor;

  let query = 'SELECT * FROM products';
  const params = [];

  if (cursor) {
    const decoded = decodeCursor(cursor);
    query += ' WHERE id > ?';
    params.push(decoded.id);
  }

  query += ' ORDER BY id ASC LIMIT ?';
  params.push(limit + 1);

  const products = await db.query(query, params);
  const hasMore = products.length > limit;
  const items = products.slice(0, limit);
  const nextCursor = hasMore ? encodeCursor({ id: items[items.length - 1].id }) : null;

  res.json({
    items,
    pagination: { nextCursor, hasMore }
  });
});
```

## Index Requirements

```sql
-- Single field cursor (ID-based)
CREATE INDEX idx_products_id ON products(id);

-- Timestamp cursor (feed-based)
CREATE INDEX idx_posts_created_at ON posts(created_at DESC, id DESC);

-- Multi-field cursor (complex sort)
CREATE INDEX idx_products_price_id ON products(price ASC, id ASC);
```
