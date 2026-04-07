---
category: Database
tags:
  - redis
  - caching
  - performance
  - interview
date: 2026-04-06T00:00:00.000Z
sections:
  - id: solution
    label: Solution
    icon: Settings
    order: 1
  - id: comparison
    label: Strategy Comparison
    icon: BarChart3
    order: 2
  - id: source
    label: Documentation
    icon: BookOpen
    order: 99
---

Your Redis cache has a 95% hit rate but during peak traffic, a popular cache key expires and 10,000 requests simultaneously hit the database. How do you prevent cache stampede?
