## JSONB Storage & Basic Queries

```sql
-- Table with flexible JSON metadata
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    name TEXT,
    metadata JSONB DEFAULT '{}'::jsonb
);

-- Sample data
INSERT INTO users (name, metadata) VALUES
('Alice', '{"age": 30, "city": "NYC", "tags": ["vip", "premium"]}'),
('Bob', '{"age": 25, "city": "LA", "tags": ["free"]}');

-- Basic queries (slow without index)
SELECT * FROM users WHERE metadata->>'age' = '30';
SELECT * FROM users WHERE metadata @> '{"city": "NYC"}';  -- Containment
SELECT * FROM users WHERE metadata ? 'age';  -- Key exists
```

## GIN Index (Best for JSONB)

```sql
-- Full JSONB index (all keys/values)
CREATE INDEX idx_users_metadata ON users USING gin(metadata);

-- Now queries are fast (index lookup)
EXPLAIN ANALYZE
SELECT * FROM users WHERE metadata @> '{"city": "NYC"}';
-- Index Scan using idx_users_metadata (< 1ms)

-- Specific key index (more efficient)
CREATE INDEX idx_users_city ON users USING gin((metadata -> 'city'));

-- Faster for this specific query
EXPLAIN ANALYZE
SELECT * FROM users WHERE metadata -> 'city' = '"NYC"'::jsonb;
```

## Expression Indexes for Specific Fields

```sql
-- Index on extracted value (avoids type conversion)
CREATE INDEX idx_users_age ON users
USING btree((metadata->>'age')::INTEGER);

-- Fast range queries
SELECT * FROM users WHERE (metadata->>'age')::INTEGER > 25;

-- Index on nested values
CREATE INDEX idx_users_plan ON users
USING btree((metadata->'subscription'->>'plan'));

SELECT * FROM users
WHERE metadata->'subscription'->>'plan' = 'premium';
```

## Array Operations in JSONB

```sql
-- JSONB array stored in metadata
INSERT INTO users (name, metadata) VALUES
('Charlie', '{"tags": ["vip", "early-adopter"]}');

-- Check if array contains value
SELECT * FROM users WHERE metadata->'tags' @> '"vip"'::jsonb;

-- Index for array containment
CREATE INDEX idx_users_tags ON users USING gin(metadata->'tags');

-- Fast array search
EXPLAIN ANALYZE
SELECT * FROM users WHERE metadata->'tags' @> '"vip"'::jsonb;
-- Index Scan using idx_users_tags
```

## Efficient Multi-Field Queries

```sql
-- Complex query
SELECT * FROM users
WHERE metadata->>'city' = 'NYC'
  AND (metadata->>'age')::INTEGER > 25
  AND metadata->'tags' @> '"premium"'::jsonb;

-- Multiple expression indexes
CREATE INDEX idx_users_city ON users USING btree((metadata->>'city'));
CREATE INDEX idx_users_age ON users USING btree(((metadata->>'age')::INTEGER));
CREATE INDEX idx_users_tags ON users USING gin((metadata->'tags'));

-- Now all conditions use indexes
EXPLAIN ANALYZE
SELECT * FROM users
WHERE metadata->>'city' = 'NYC'
  AND (metadata->>'age')::INTEGER > 25
  AND metadata->'tags' @> '"premium"'::jsonb;
-- Bitmap Index Scan + merge (very fast)
```

## Full-Text Search in JSONB

```sql
-- Store profile + description
UPDATE users SET metadata = metadata || '{
  "profile": "Senior engineer with 10 years experience",
  "interests": ["cloud", "databases", "golang"]
}'::jsonb;

-- Create tsvector index on JSONB text
CREATE INDEX idx_users_profile_search ON users USING gin(
    to_tsvector('english', metadata->>'profile')
);

-- Full-text search within JSONB
SELECT * FROM users, plainto_tsquery('english', 'databases') as query
WHERE to_tsvector('english', metadata->>'profile') @@ query;
```

## Real Example: E-Commerce Order Metadata

```sql
CREATE TABLE orders (
    id SERIAL PRIMARY KEY,
    customer_id INTEGER,
    total DECIMAL,
    metadata JSONB  -- Flexible order metadata
);

-- Insert orders with various metadata
INSERT INTO orders (customer_id, total, metadata) VALUES
(1, 150.00, '{
    "items": 3,
    "status": "shipped",
    "shipping_address": {"city": "NYC", "zip": "10001"},
    "payment_method": "credit_card",
    "tags": ["gift", "urgent"]
}'::jsonb),
(2, 89.99, '{
    "items": 1,
    "status": "pending",
    "shipping_address": {"city": "LA", "zip": "90001"},
    "payment_method": "paypal",
    "tags": ["sales"]
}'::jsonb);

-- Indexes for common queries
CREATE INDEX idx_orders_status ON orders USING btree((metadata->>'status'));
CREATE INDEX idx_orders_city ON orders USING btree((metadata->'shipping_address'->>'city'));
CREATE INDEX idx_orders_tags ON orders USING gin((metadata->'tags'));
CREATE INDEX idx_orders_items ON orders USING btree(((metadata->>'items')::INTEGER));

-- Fast queries
SELECT * FROM orders WHERE metadata->>'status' = 'shipped';
SELECT * FROM orders WHERE metadata->'shipping_address'->>'city' = 'NYC';
SELECT * FROM orders WHERE metadata->'tags' @> '"gift"'::jsonb;
SELECT * FROM orders WHERE (metadata->>'items')::INTEGER > 2;
```

## Comparison of JSONB vs Columns

```
SCENARIO: Store product metadata (100M rows)

OPTION 1: JSONB Metadata Column
  CREATE TABLE products (
    id INT, name TEXT,
    metadata JSONB  -- {color, size, brand, warranty, rating}
  );
  CREATE INDEX idx_metadata ON products USING gin(metadata);

  Speed: Index scan ~5-20ms for specific filter
  Pros: Flexible, can add fields anytime
  Cons: Slight overhead vs native columns

OPTION 2: Native Columns
  CREATE TABLE products (
    id INT, name TEXT,
    color TEXT, size TEXT, brand TEXT, warranty INTEGER, rating DECIMAL
  );
  CREATE INDEX idx_color ON products(color);
  CREATE INDEX idx_rating ON products(rating);

  Speed: Same or slightly faster (~1-5ms)
  Pros: Familiar, optimized
  Cons: Rigid, altering table is slow (schema migration)

RECOMMENDATION:
- If fields are fixed (90% of them): Use native columns
- If fields change frequently: Use JSONB
- Hybrid: Core fields as columns, extras in JSONB
```

## Partial JSONB Index

```sql
-- Index only orders from specific region
CREATE INDEX idx_orders_ny_status ON orders USING btree((metadata->>'status'))
WHERE metadata->'shipping_address'->>'city' = 'NYC';

-- Index applies only to NY orders
SELECT * FROM orders
WHERE metadata->'shipping_address'->>'city' = 'NYC'
  AND metadata->>'status' = 'shipped';
-- Much smaller index (fast)

-- Doesn't apply to other cities
SELECT * FROM orders
WHERE metadata->'shipping_address'->>'city' = 'LA'
  AND metadata->>'status' = 'shipped';
-- Full scan, but smaller dataset
```
