## tsvector vs pg_trgm vs Elasticsearch

| Feature | tsvector | pg_trgm | Elasticsearch |
|---------|----------|---------|---------------|
| **Setup Time** | 20 min | 5 min | 1 hour |
| **Storage** | +10% disk | +30% disk | Separate cluster |
| **Query Speed** | 10-50ms | 50-200ms | 1-10ms |
| **Typo Tolerance** | No | **Yes** | **Yes** |
| **Language Support** | 15+ languages | Language-agnostic | Multiple analyzers |
| **Stemming** | Built-in | No | Built-in |
| **Phrase Search** | Yes | No | **Yes** |
| **Fuzzy Matching** | No | **Yes (similarity)** | **Yes** |
| **Scaling** | Single DB | Single DB | Horizontal |
| **Complexity** | Medium | Low | High |
| **Operational Overhead** | Low | Low | Medium-High |

## Decision Tree

```
Do you need typo/fuzzy matching?
├─ YES
│  ├─ Scale: Single server?
│  │  ├─ YES → Use pg_trgm (simple, effective)
│  │  └─ NO → Use Elasticsearch (distributed)
│  └─ Want weighted relevance?
│     ├─ YES → Combine tsvector + pg_trgm
│     └─ NO → pg_trgm alone
│
└─ NO (exact word matching only)
   ├─ Scale: Single server?
   │  ├─ YES → Use tsvector (fastest, simplest)
   │  └─ NO → Use Elasticsearch (distributed)
   └─ Need phrase search?
      ├─ YES → tsvector
      └─ NO → tsvector
```

## Use Case Examples

### Scenario 1: E-commerce Product Search (100M items)
**Requirement:** Typo-tolerant, fast, must scale, 99.9% uptime

**Solution: Elasticsearch**
- Handles 1M+ QPS with typos
- Distributed across nodes
- Auto-failover, replication
- Complex analytics/faceting

```bash
# Elasticsearch setup
docker run -e "discovery.type=single-node" elastic/elasticsearch:8.0.0

# Product indexing
PUT /products
{
  "mappings": {
    "properties": {
      "name": { "type": "text", "analyzer": "standard" },
      "description": { "type": "text" },
      "tags": { "type": "keyword" }
    }
  }
}

# Fuzzy search with typos
GET /products/_search
{
  "query": {
    "multi_match": {
      "query": "lapto",
      "fields": ["name^2", "description"],
      "fuzziness": "AUTO"
    }
  }
}
```

### Scenario 2: Internal Wiki Search (10M documents)
**Requirement:** Typo tolerance, low cost, single-server

**Solution: pg_trgm**
- Lowest overhead
- Handles typos natively
- Leverage existing PostgreSQL

```sql
CREATE EXTENSION pg_trgm;
CREATE INDEX idx_wiki_content_trgm ON wiki_pages USING gist(content gist_trgm_ops);

SELECT id, title, content
FROM wiki_pages
WHERE content % 'postgresq'  -- Typo in "postgresql"
ORDER BY similarity(content, 'postgresq') DESC;
```

### Scenario 3: Blog Post Search (100K posts)
**Requirement:** Weighted ranking, phrase search, no typos, single server

**Solution: tsvector + Ranking**
- Fast (GIN index)
- Excellent relevance (ranking)
- Language support (English, French, etc.)

```sql
CREATE INDEX idx_blog_search ON blog_posts USING gin(search_vector);

-- Weighted search (title > content)
SELECT id, title,
       ts_rank_cd(search_vector, query) as relevance
FROM blog_posts, plainto_tsquery('english', 'postgresql performance') as query
WHERE search_vector @@ query
ORDER BY relevance DESC;
```

### Scenario 4: User Profile Search in SaaS (1M users)
**Requirement:** Name + email search, typo-tolerant, real-time

**Solution: Hybrid (tsvector + pg_trgm)**
```sql
-- Exact name + email indexed
CREATE INDEX idx_users_search ON users USING gin(search_vector);

-- Fuzzy for typos
CREATE INDEX idx_users_name_trgm ON users USING gist(name gist_trgm_ops);

-- Smart query
SELECT id, name, email FROM users
WHERE search_vector @@ plainto_tsquery('english', 'john')
   OR name % 'john'
   OR email % 'john@exampl.com'
ORDER BY CASE
    WHEN name ILIKE 'john%' THEN 0
    WHEN search_vector @@ plainto_tsquery('english', 'john') THEN 1
    ELSE 2
END;
```

## Performance Testing

```python
import time
import psycopg2

queries = [
    "laptop",
    "lapto",  # typo
    "gaming computer",
    "gamin computr",  # multiple typos
]

for method in ['like', 'tsvector', 'pg_trgm', 'combined']:
    total_time = 0
    for query in queries * 100:  # Run 100 times each
        start = time.time()

        # Execute query based on method
        if method == 'like':
            cursor.execute(
                "SELECT * FROM products WHERE name LIKE %s OR description LIKE %s",
                (f'%{query}%', f'%{query}%')
            )
        elif method == 'tsvector':
            cursor.execute(
                "SELECT * FROM products WHERE search_vector @@ plainto_tsquery(%s)",
                (query,)
            )
        elif method == 'pg_trgm':
            cursor.execute(
                "SELECT * FROM products WHERE name % %s ORDER BY similarity(name, %s)",
                (query, query)
            )
        # ... etc

        total_time += time.time() - start

    print(f"{method}: {total_time/len(queries):.2f}ms avg")

# Output:
# like: 2500.45ms avg (SLOW, no typos)
# tsvector: 12.34ms avg (FAST, no typos)
# pg_trgm: 45.67ms avg (GOOD, typo-tolerant)
# combined: 15.23ms avg (BEST, typo + speed)
```

## Migration from LIKE to tsvector

```sql
-- Step 1: Add tsvector column
ALTER TABLE products ADD COLUMN search_vector tsvector;

-- Step 2: Backfill in batches
DO $$
BEGIN
    LOOP
        UPDATE products
        SET search_vector = to_tsvector('english', name || ' ' || coalesce(description, ''))
        WHERE search_vector IS NULL
        LIMIT 10000;
        EXIT WHEN NOT FOUND;
        PERFORM pg_sleep(1);  -- Pace the update
    END LOOP;
END $$;

-- Step 3: Create index
CREATE INDEX idx_products_search ON products USING gin(search_vector);

-- Step 4: Add trigger for future updates
CREATE TRIGGER trigger_products_search
BEFORE INSERT OR UPDATE ON products
FOR EACH ROW
EXECUTE FUNCTION products_update_search();

-- Step 5: Update application code
-- Old: WHERE name LIKE '%term%'
-- New: WHERE search_vector @@ plainto_tsquery('english', 'term')

-- Step 6: Monitor and drop old LIKE indexes (after verification)
-- DROP INDEX idx_products_name;
```
