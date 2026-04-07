## The Problem: LIKE is Slow

```sql
-- Slow: full table scan even with index
SELECT * FROM products
WHERE name LIKE '%laptop%' OR description LIKE '%laptop%';
-- Sequential scan on 100M rows (5+ seconds)

CREATE INDEX idx_products_name ON products(name);
-- LIKE '%text%' still can't use index (begins with wildcard)
```

## Solution 1: PostgreSQL Full-Text Search (tsvector)

Best for: Built-in, reasonable language support, single-server.

```sql
-- Step 1: Add tsvector column to store searchable text
ALTER TABLE products ADD COLUMN search_vector tsvector;

-- Step 2: Index it with GIN (best for text search)
CREATE INDEX idx_products_search ON products USING gin(search_vector);

-- Step 3: Create trigger to auto-update search_vector
CREATE OR REPLACE FUNCTION products_update_search()
RETURNS TRIGGER AS $$
BEGIN
    NEW.search_vector := to_tsvector('english',
        coalesce(NEW.name, '') || ' ' ||
        coalesce(NEW.description, '') || ' ' ||
        coalesce(array_to_string(NEW.tags, ' '), '')
    );
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trigger_products_search
BEFORE INSERT OR UPDATE ON products
FOR EACH ROW
EXECUTE FUNCTION products_update_search();

-- Step 4: Query with @@ operator (super fast)
SELECT id, name, ts_rank(search_vector, query) as rank
FROM products, plainto_tsquery('english', 'laptop computer') as query
WHERE search_vector @@ query
ORDER BY rank DESC
LIMIT 20;
-- GIN index lookup: <50ms on 100M rows

-- Backfill existing data
UPDATE products SET search_vector = to_tsvector('english',
    coalesce(name, '') || ' ' || coalesce(description, '')
);
```

**Advanced: Weighted search (title more important than description)**
```sql
CREATE OR REPLACE FUNCTION products_update_search()
RETURNS TRIGGER AS $$
BEGIN
    NEW.search_vector :=
        setweight(to_tsvector('english', coalesce(NEW.name, '')), 'A') ||
        setweight(to_tsvector('english', coalesce(NEW.description, '')), 'B') ||
        setweight(to_tsvector('english', coalesce(array_to_string(NEW.tags, ' '), '')), 'C');
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

-- Ranking prefers A (title) matches
SELECT id, name, ts_rank_cd(search_vector, query) as rank
FROM products, plainto_tsquery('english', 'laptop') as query
WHERE search_vector @@ query
ORDER BY rank DESC;
```

## Solution 2: Trigram Search (pg_trgm)

Best for: Typo tolerance, fuzzy matching, simple implementation.

```sql
-- Enable extension
CREATE EXTENSION IF NOT EXISTS pg_trgm;

-- Create GiST or GIST index for similarity
CREATE INDEX idx_products_name_trgm ON products USING gist(name gist_trgm_ops);

-- Query with % (similarity) operator
SELECT id, name, similarity(name, 'lapto') as score
FROM products
WHERE name % 'lapto'  -- Matches 'laptop' (typo tolerance)
ORDER BY score DESC
LIMIT 20;

-- Increase similarity threshold for stricter matches
SET pg_trgm.similarity_threshold = 0.6;

SELECT * FROM products WHERE name % 'lamtop';  -- Matches 'laptop'
```

**Multi-column fuzzy search:**
```sql
-- Index on multiple columns
CREATE INDEX idx_products_multi_trgm ON products
USING gist((name || ' ' || description) gist_trgm_ops);

SELECT id, name, description,
       similarity(name || ' ' || description, 'lapto computer') as score
FROM products
WHERE (name || ' ' || description) % 'lapto computer'
ORDER BY score DESC
LIMIT 20;
```

## Solution 3: Combined Approach (Best of Both)

Use **tsvector for exact words** + **pg_trgm for typos**:

```sql
CREATE EXTENSION IF NOT EXISTS pg_trgm;

-- Both indexes
CREATE INDEX idx_products_search ON products USING gin(search_vector);
CREATE INDEX idx_products_trgm ON products USING gist(name gist_trgm_ops);

-- Search function that tries both
CREATE OR REPLACE FUNCTION search_products(query text)
RETURNS TABLE(id int, name text, description text, rank float8) AS $$
BEGIN
    RETURN QUERY
    SELECT p.id, p.name, p.description,
           CASE
               WHEN p.search_vector @@ plainto_tsquery('english', query)
               THEN 2.0  -- Exact word match scores higher
               WHEN p.name % query
               THEN 1.0 + similarity(p.name, query)  -- Typo match
               ELSE similarity(p.name || ' ' || p.description, query)
           END as rank
    FROM products p
    WHERE p.search_vector @@ plainto_tsquery('english', query)
       OR p.name % query
       OR (p.name || ' ' || p.description) % query
    ORDER BY rank DESC
    LIMIT 20;
END;
$$ LANGUAGE plpgsql;

-- Usage
SELECT * FROM search_products('lapto');  -- Handles typo
SELECT * FROM search_products('laptop computer');  -- Exact search
```

## Real Example: E-Commerce Search

```sql
-- Product table
CREATE TABLE products (
    id SERIAL PRIMARY KEY,
    name TEXT NOT NULL,
    description TEXT,
    category TEXT,
    price DECIMAL,
    tags TEXT[],
    search_vector tsvector,
    created_at TIMESTAMP DEFAULT now()
);

-- Composite search function
CREATE OR REPLACE FUNCTION search_products(
    search_term text,
    category_filter text DEFAULT NULL,
    max_price DECIMAL DEFAULT NULL
)
RETURNS TABLE(id int, name text, description text, price decimal, rank float8) AS $$
DECLARE
    tsquery_obj tsquery;
BEGIN
    tsquery_obj := plainto_tsquery('english', search_term);

    RETURN QUERY
    SELECT
        p.id,
        p.name,
        p.description,
        p.price,
        CASE
            WHEN p.name ILIKE search_term || '%' THEN 3.0
            WHEN p.search_vector @@ tsquery_obj THEN 2.0
            WHEN p.name % search_term THEN 1.0 + similarity(p.name, search_term)
            ELSE similarity(p.description, search_term)
        END as rank
    FROM products p
    WHERE
        (p.search_vector @@ tsquery_obj OR p.name % search_term)
        AND (category_filter IS NULL OR p.category = category_filter)
        AND (max_price IS NULL OR p.price <= max_price)
    ORDER BY rank DESC
    LIMIT 50;
END;
$$ LANGUAGE plpgsql;

-- Usage
SELECT * FROM search_products('laptop', 'electronics', 1500);
SELECT * FROM search_products('gaming lapto', 'electronics');  -- Typo OK
```

## Benchmark: Speed Comparison

| Method | 1M Rows | 100M Rows | Typo Support | Setup |
|--------|---------|-----------|--------------|-------|
| LIKE '%term%' | 500ms | 30s+ | No | Trivial |
| LIKE with Index | 100ms | 5s+ | No | Needs prefix |
| tsvector + GIN | 10ms | 50ms | No | Medium |
| pg_trgm + GiST | 50ms | 200ms | **Yes** | Medium |
| Combined | 10ms | 50ms | **Yes** | Complex |

