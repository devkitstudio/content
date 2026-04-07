# Elasticsearch Search Architecture

## Why Database Search Fails

```
SELECT * FROM products
WHERE title LIKE '%laptop%'
  AND price BETWEEN 500 AND 2000
  AND category = 'Electronics'
ORDER BY relevance, price
LIMIT 20;

Problems:
- LIKE '%text%' is O(n) full table scan
- No fuzzy matching (typos: "lapotop")
- No relevance ranking (BM25)
- Facets require GROUP BY (slow)
- No autocomplete
```

## Architecture

```
┌────────────────────────────────────────┐
│   Product Catalog (PostgreSQL)         │
│   - Source of truth                    │
│   - ID, name, description, price       │
└────────────┬─────────────────────────┘
             │
             │ (Sync on update)
             │ (Stream: Kafka/Logstash)
             ▼
┌────────────────────────────────────────┐
│   Elasticsearch Index                  │
│   - Optimized for search               │
│   - Inverted index                     │
│   - Facets (aggregations)              │
│   - Autocomplete (n-grams)             │
└────────────┬─────────────────────────┘
             │
             ▲
             │ (Query)
             │
┌────────────────────────────────────────┐
│   Search API (FastAPI/Express)         │
│   - Parse user query                   │
│   - Build Elasticsearch query          │
│   - Return results + facets            │
└────────────────────────────────────────┘
```

## Index Design

```python
PUT /products
{
  "settings": {
    "number_of_shards": 5,
    "number_of_replicas": 1,
    "analysis": {
      "analyzer": {
        "default_analyzer": {
          "type": "custom",
          "tokenizer": "standard",
          "filter": ["lowercase", "stop", "snowball"]
        },
        "autocomplete_analyzer": {
          "type": "custom",
          "tokenizer": "standard",
          "filter": ["lowercase", "edge_ngram_filter"]
        }
      },
      "filter": {
        "edge_ngram_filter": {
          "type": "edge_ngram",
          "min_gram": 2,
          "max_gram": 20
        }
      }
    }
  },
  "mappings": {
    "properties": {
      "id": {
        "type": "keyword"
      },
      "title": {
        "type": "text",
        "analyzer": "default_analyzer",
        "fields": {
          "keyword": {
            "type": "keyword"
          },
          "autocomplete": {
            "type": "text",
            "analyzer": "autocomplete_analyzer"
          }
        }
      },
      "description": {
        "type": "text",
        "analyzer": "default_analyzer"
      },
      "price": {
        "type": "float"
      },
      "category": {
        "type": "keyword"
      },
      "tags": {
        "type": "keyword"
      },
      "brand": {
        "type": "keyword"
      },
      "rating": {
        "type": "float"
      },
      "created_at": {
        "type": "date"
      },
      "popularity_score": {
        "type": "float"
      }
    }
  }
}
```

**Key fields:**
- `title.autocomplete`: Prefix matching for autocomplete
- `category, brand, tags`: Keywords for filtering (no analysis)
- `price`: Range queries
- `rating, popularity_score`: For sorting

## Search Query Patterns

### 1. Full-Text Search + Filters + Sorting

```
User: "gaming laptop under $2000"
```

```python
GET /products/_search
{
  "query": {
    "bool": {
      "must": [
        {
          "multi_match": {
            "query": "gaming laptop",
            "fields": ["title^2", "description"],
            "fuzziness": "AUTO"  # Handle typos
          }
        }
      ],
      "filter": [
        {
          "range": {
            "price": {
              "gte": 0,
              "lte": 2000
            }
          }
        },
        {
          "terms": {
            "category": ["Electronics", "Computers"]
          }
        }
      ]
    }
  },
  "sort": [
    { "popularity_score": { "order": "desc" } },
    { "_score": { "order": "desc" } }
  ],
  "size": 20,
  "from": 0,
  "aggs": {
    "brands": {
      "terms": {
        "field": "brand",
        "size": 10
      }
    },
    "price_ranges": {
      "range": {
        "field": "price",
        "ranges": [
          { "to": 500 },
          { "from": 500, "to": 1000 },
          { "from": 1000, "to": 2000 },
          { "from": 2000 }
        ]
      }
    },
    "rating": {
      "range": {
        "field": "rating",
        "ranges": [
          { "from": 4 },
          { "from": 3, "to": 4 },
          { "from": 2, "to": 3 }
        ]
      }
    }
  }
}
```

### 2. Autocomplete

```
User types: "game"
```

```python
GET /products/_search
{
  "query": {
    "match": {
      "title.autocomplete": {
        "query": "game",
        "analyzer": "autocomplete_analyzer"
      }
    }
  },
  "size": 10,
  "_source": ["id", "title"]
}
```

### 3. Faceted Search

```python
# Returns:
{
  "took": 15,
  "hits": {
    "total": { "value": 1200 },
    "hits": [...]  # Results
  },
  "aggregations": {
    "brands": {
      "buckets": [
        { "key": "Dell", "doc_count": 450 },
        { "key": "HP", "doc_count": 380 },
        { "key": "Apple", "doc_count": 370 }
      ]
    },
    "price_ranges": {
      "buckets": [
        { "key": "*-500", "doc_count": 200 },
        { "key": "500-1000", "doc_count": 350 },
        { "key": "1000-2000", "doc_count": 450 },
        { "key": "2000-*", "doc_count": 200 }
      ]
    }
  }
}
```

## Sync Pipeline

### Option 1: Logstash Pipeline (Simple)

```conf
# logstash.conf
input {
  jdbc {
    jdbc_connection_string => "jdbc:postgresql://localhost:5432/shop"
    jdbc_user => "user"
    jdbc_password => "pass"
    jdbc_driver_library => "/path/to/postgresql-42.jar"
    jdbc_driver_class => "org.postgresql.Driver"
    statement => "SELECT * FROM products WHERE updated_at > :sql_last_value"
    schedule => "*/5 * * * *"  # Every 5 minutes
    last_run_metadata_path => "/path/to/metadata"
  }
}

filter {
  mutate {
    convert => {
      "price" => "float"
      "rating" => "float"
    }
  }
}

output {
  elasticsearch {
    hosts => ["localhost:9200"]
    index => "products"
    document_id => "%{id}"
  }
}
```

### Option 2: Event-Driven (Better)

```python
# Producer: When product updates, emit event
from kafka import KafkaProducer

producer = KafkaProducer(bootstrap_servers=['localhost:9092'])

# When product is updated
def update_product(product_id, data):
    db.products.update_one({'id': product_id}, {'$set': data})

    # Emit event for search indexing
    producer.send('search.products.updated', value={
        'event': 'product_updated',
        'product_id': product_id,
        'timestamp': datetime.utcnow().isoformat()
    })

# Consumer: Index into Elasticsearch
from elasticsearch import Elasticsearch

es = Elasticsearch(['localhost:9200'])
consumer = KafkaConsumer('search.products.updated')

for message in consumer:
    event = json.loads(message.value)
    product_id = event['product_id']

    # Fetch from DB
    product = db.products.find_one({'id': product_id})

    # Index into ES
    es.index(
        index='products',
        id=product_id,
        document={
            'id': product['id'],
            'title': product['title'],
            'description': product['description'],
            'price': product['price'],
            'category': product['category'],
            'tags': product['tags'],
            'brand': product['brand'],
            'rating': product['rating'],
            'popularity_score': calculate_popularity(product)
        }
    )
```

## Performance Tips

1. **Index sizing:** 1 shard per 30GB data
2. **Replicas:** 1-2 for redundancy
3. **Refresh interval:** `30s` (default) balances freshness vs performance
4. **Query timeout:** Set to 1s to catch slow queries

```python
# Slow query configuration
PUT /products/_settings
{
  "index.search.slowlog.threshold.query.warn": "1s",
  "index.search.slowlog.threshold.query.info": "500ms",
  "index.search.slowlog.threshold.query.debug": "100ms"
}
```

5. **Caching:** Cache popular searches

```python
from functools import lru_cache

@lru_cache(maxsize=1000)
def search_products(query: str, filters: dict):
    return es.search(index='products', body=build_query(query, filters))
```

## Fallback Strategy

If Elasticsearch is down:
1. Query database directly (slow but works)
2. Use read cache if available
3. Return "search temporarily unavailable" with cached results

```python
def search_products(query: str, filters: dict):
    try:
        return es.search(index='products', body=build_query(query, filters))
    except Exception as e:
        logger.error(f"ES search failed: {e}")

        # Fallback to database
        return fallback_db_search(query, filters)
```

## Monitoring

```python
# Health check
es_health = es.cluster.health()
# {
#   "status": "green",  # green = all shards assigned
#   "number_of_nodes": 3,
#   "active_shards": 10
# }

# Indexing rate
stats = es.indices.stats(index='products')
stats['indices']['products']['primaries']['indexing']['index_total']  # Total docs indexed
```
