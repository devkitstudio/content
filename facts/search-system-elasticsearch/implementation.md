# Mapping + Query DSL Examples

## Complete Product Index Mapping

```python
from elasticsearch import Elasticsearch
import json

es = Elasticsearch(['localhost:9200'])

# Create index with custom mapping
index_mapping = {
    "settings": {
        "number_of_shards": 5,
        "number_of_replicas": 1,
        "analysis": {
            "tokenizer": {
                "product_tokenizer": {
                    "type": "pattern",
                    "pattern": "[\\W]+"
                }
            },
            "filter": {
                "english_stop": {
                    "type": "stop",
                    "stopwords": "_english_"
                },
                "english_stemmer": {
                    "type": "stemmer",
                    "language": "english"
                },
                "edge_ngram_filter": {
                    "type": "edge_ngram",
                    "min_gram": 2,
                    "max_gram": 20
                }
            },
            "analyzer": {
                "product_search": {
                    "type": "custom",
                    "tokenizer": "standard",
                    "filter": [
                        "lowercase",
                        "english_stop",
                        "english_stemmer"
                    ]
                },
                "autocomplete": {
                    "type": "custom",
                    "tokenizer": "standard",
                    "filter": [
                        "lowercase",
                        "edge_ngram_filter"
                    ]
                }
            }
        }
    },
    "mappings": {
        "properties": {
            "id": {
                "type": "keyword"
            },
            "sku": {
                "type": "keyword"
            },
            "title": {
                "type": "text",
                "analyzer": "product_search",
                "boost": 2.0,
                "fields": {
                    "raw": {
                        "type": "keyword"
                    },
                    "autocomplete": {
                        "type": "text",
                        "analyzer": "autocomplete"
                    }
                }
            },
            "description": {
                "type": "text",
                "analyzer": "product_search"
            },
            "brand": {
                "type": "keyword"
            },
            "category": {
                "type": "keyword"
            },
            "subcategory": {
                "type": "keyword"
            },
            "tags": {
                "type": "keyword"
            },
            "price": {
                "type": "float"
            },
            "discount_price": {
                "type": "float"
            },
            "rating": {
                "type": "float"
            },
            "review_count": {
                "type": "integer"
            },
            "in_stock": {
                "type": "boolean"
            },
            "stock_count": {
                "type": "integer"
            },
            "created_at": {
                "type": "date"
            },
            "updated_at": {
                "type": "date"
            },
            "popularity_score": {
                "type": "float"
            },
            "color": {
                "type": "keyword"
            },
            "size": {
                "type": "keyword"
            }
        }
    }
}

# Create the index
es.indices.create(index='products', body=index_mapping)
```

## Complex Query Examples

### 1. Product Search with All Filters

```python
def build_search_query(
    search_text: str = None,
    price_min: float = None,
    price_max: float = None,
    brands: list = None,
    categories: list = None,
    min_rating: float = None,
    in_stock_only: bool = False,
    sort_by: str = "relevance",
    page: int = 1
):
    """Build a complex Elasticsearch query"""

    must_clauses = []
    filter_clauses = []

    # Full-text search on title + description
    if search_text:
        must_clauses.append({
            "multi_match": {
                "query": search_text,
                "fields": [
                    "title^3",  # Title weighted 3x
                    "description^1"
                ],
                "type": "best_fields",
                "operator": "or",
                "fuzziness": "AUTO",
                "max_expansions": 50
            }
        })

    # Price range filter
    if price_min is not None or price_max is not None:
        range_query = {}
        if price_min is not None:
            range_query["gte"] = price_min
        if price_max is not None:
            range_query["lte"] = price_max

        filter_clauses.append({
            "range": {
                "price": range_query
            }
        })

    # Brand filter
    if brands:
        filter_clauses.append({
            "terms": {
                "brand": brands
            }
        })

    # Category filter
    if categories:
        filter_clauses.append({
            "terms": {
                "category": categories
            }
        })

    # Rating filter
    if min_rating is not None:
        filter_clauses.append({
            "range": {
                "rating": {
                    "gte": min_rating
                }
            }
        })

    # Stock filter
    if in_stock_only:
        filter_clauses.append({
            "term": {
                "in_stock": True
            }
        })

    # Build the bool query
    query_body = {
        "query": {
            "bool": {
                "must": must_clauses if must_clauses else [{"match_all": {}}],
                "filter": filter_clauses
            }
        },
        "size": 20,
        "from": (page - 1) * 20
    }

    # Sorting
    sort_options = {
        "relevance": [
            {"_score": {"order": "desc"}},
            {"popularity_score": {"order": "desc"}}
        ],
        "price_low": [{"price": {"order": "asc"}}],
        "price_high": [{"price": {"order": "desc"}}],
        "newest": [{"created_at": {"order": "desc"}}],
        "rating": [
            {"rating": {"order": "desc"}},
            {"review_count": {"order": "desc"}}
        ]
    }

    query_body["sort"] = sort_options.get(sort_by, sort_options["relevance"])

    # Add aggregations (facets)
    query_body["aggs"] = {
        "brands": {
            "terms": {
                "field": "brand",
                "size": 10
            }
        },
        "categories": {
            "terms": {
                "field": "category",
                "size": 20
            }
        },
        "price_ranges": {
            "range": {
                "field": "price",
                "ranges": [
                    {"to": 50},
                    {"from": 50, "to": 100},
                    {"from": 100, "to": 200},
                    {"from": 200, "to": 500},
                    {"from": 500, "to": 1000},
                    {"from": 1000}
                ]
            }
        },
        "rating_distribution": {
            "range": {
                "field": "rating",
                "ranges": [
                    {"to": 2},
                    {"from": 2, "to": 3},
                    {"from": 3, "to": 4},
                    {"from": 4, "to": 5},
                    {"from": 5}
                ]
            }
        }
    }

    return query_body


# Execute search
def search(search_params):
    query = build_search_query(**search_params)
    results = es.search(index='products', body=query)

    return {
        "total": results['hits']['total']['value'],
        "products": [
            {
                "id": hit['_id'],
                "score": hit['_score'],
                **hit['_source']
            }
            for hit in results['hits']['hits']
        ],
        "facets": {
            "brands": [
                {
                    "name": bucket['key'],
                    "count": bucket['doc_count']
                }
                for bucket in results['aggregations']['brands']['buckets']
            ],
            "price_ranges": [
                {
                    "range": f"{bucket.get('from', 0)}-{bucket.get('to', 'inf')}",
                    "count": bucket['doc_count']
                }
                for bucket in results['aggregations']['price_ranges']['buckets']
            ]
        }
    }
```

### 2. Autocomplete Query

```python
def autocomplete(prefix: str, category: str = None):
    """Product autocomplete"""
    must_clauses = [
        {
            "match": {
                "title.autocomplete": {
                    "query": prefix,
                    "analyzer": "autocomplete"
                }
            }
        }
    ]

    filter_clauses = []
    if category:
        filter_clauses.append({
            "term": {"category": category}
        })

    query = {
        "query": {
            "bool": {
                "must": must_clauses,
                "filter": filter_clauses
            }
        },
        "size": 10,
        "_source": ["id", "title", "brand", "price"],
        "highlight": {
            "fields": {
                "title": {}
            }
        }
    }

    results = es.search(index='products', body=query)

    return [
        {
            "id": hit['_id'],
            "title": hit.get('highlight', {}).get('title', [hit['_source']['title']])[0],
            "brand": hit['_source'].get('brand'),
            "price": hit['_source'].get('price')
        }
        for hit in results['hits']['hits']
    ]
```

### 3. More Like This Query

```python
def more_like_this(product_id: str):
    """Find similar products"""
    query = {
        "query": {
            "more_like_this": {
                "fields": ["title", "description", "tags", "category"],
                "like": [
                    {
                        "_index": "products",
                        "_id": product_id
                    }
                ],
                "min_term_freq": 2,
                "max_query_terms": 20
            }
        },
        "size": 5
    }

    results = es.search(index='products', body=query)
    return results['hits']['hits']
```

## Bulk Indexing

```python
from elasticsearch.helpers import bulk

def bulk_index_products(products: list):
    """Efficiently index multiple products"""

    actions = [
        {
            "_index": "products",
            "_id": product['id'],
            "_source": {
                "id": product['id'],
                "title": product['title'],
                "description": product['description'],
                "price": product['price'],
                "category": product['category'],
                "brand": product['brand'],
                "rating": product['rating'],
                "in_stock": product['in_stock'],
                "created_at": product['created_at'],
                "popularity_score": calculate_popularity(product)
            }
        }
        for product in products
    ]

    # Bulk index
    success, failed = bulk(es, actions, chunk_size=500)
    print(f"Indexed {success} products, {len(failed)} failed")

    return failed


# Usage
products = db.products.find().limit(10000)
bulk_index_products(list(products))
```

## API Endpoint

```python
from fastapi import FastAPI, Query
from typing import Optional, List

app = FastAPI()

@app.get("/api/search")
def search_products(
    q: str = Query(None),
    price_min: Optional[float] = None,
    price_max: Optional[float] = None,
    brands: Optional[List[str]] = Query(None),
    categories: Optional[List[str]] = Query(None),
    min_rating: Optional[float] = None,
    in_stock: bool = False,
    sort_by: str = "relevance",
    page: int = 1
):
    """Search products with filters"""

    results = search({
        "search_text": q,
        "price_min": price_min,
        "price_max": price_max,
        "brands": brands or [],
        "categories": categories or [],
        "min_rating": min_rating,
        "in_stock_only": in_stock,
        "sort_by": sort_by,
        "page": page
    })

    return results


@app.get("/api/autocomplete")
def autocomplete_products(
    prefix: str = Query(...),
    category: Optional[str] = None
):
    """Autocomplete product names"""
    return autocomplete(prefix, category)
```

This design handles millions of products with sub-100ms search latency.
