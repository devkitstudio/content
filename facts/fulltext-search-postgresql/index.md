---
category: Database
tags:
  - postgresql
  - search
  - performance
  - interview
date: 2026-04-06T00:00:00.000Z
sections:
  - id: solution
    label: Solution
    icon: Settings
    order: 1
  - id: comparison
    label: Method Comparison
    icon: BarChart3
    order: 2
  - id: source
    label: Documentation
    icon: BookOpen
    order: 99
---

You need to search products by name, description, and tags with typo tolerance. PostgreSQL LIKE is too slow. How do you implement full-text search — pg_trgm vs tsvector vs Elasticsearch?
