By default, Elasticsearch limits `from` (offset) pagination to 10,000 records (the `index.max_result_window` setting). To smoothly bypass this limit on datasets with millions of rows, the `search_after` API is the absolute standard.

Fundamentally, `search_after` is **Cursor-Based Pagination** designed specifically for the distributed architecture of a Search Engine.

### The Mandatory Requirement: The Tiebreaker
To ensure `search_after` does not skip or duplicate data when records share the same sorting value (e.g., 50 products all priced at $100), you **must** provide a unique field at the end of the `sort` array. Typically, the `_id` field (or a unique UUID) serves as this tiebreaker.

### Implementation Guide (JSON DSL)

**1. Querying the First Page**
Sort the data according to business requirements (e.g., `price` descending), along with the unique tiebreaker (`_id` ascending).

```json
GET /products/_search
{
  "size": 20,
  "query": { "match": { "category": "electronics" } },
  "sort": [
    { "price": "desc" },
    { "_id": "asc" } 
  ]
}
```

**2. Extracting the Cursor**
In the response, bypass counting the total pages. Instead, simply extract the `sort` array located inside the very last object (`hit`) of the data list. This array acts as your physical Cursor.

```json
"hits": [
  // ... previous 19 records
  {
    "_index": "products",
    "_id": "doc-8910",
    "_source": { "price": 199.99, "name": "Mechanical Keyboard" },
    "sort": [199.99, "doc-8910"] // <- Extract this array
  }
]
```

**3. Querying the Next Page**
The client attaches the extracted `sort` array into the `search_after` parameter for the next request. The `sort` and `query` structures must remain exactly identical to the initial request.

```json
GET /products/_search
{
  "size": 20,
  "query": { "match": { "category": "electronics" } },
  "search_after": [199.99, "doc-8910"],
  "sort": [
    { "price": "desc" },
    { "_id": "asc" }
  ]
}
```

### The Architectural Reality (Bypassing Deep Pagination)
With `from: 100000` pagination, Elasticsearch forces **all** Shards to collect 100,020 documents, push them all over the network to a single Coordinating Node, consume RAM/CPU to merge and sort them locally, and then ruthlessly discard the first 100,000 documents.

With `search_after`, the array `[199.99, "doc-8910"]` acts as a direct coordinate pointing straight into the B-Tree (or Inverted Index). The Shards simply locate this coordinate and retrieve the 20 records immediately following it. This completely eliminates the networking, RAM, and computation overhead of "counting and discarding." The latency of fetching page 2 versus page 50,000 remains virtually identical.
