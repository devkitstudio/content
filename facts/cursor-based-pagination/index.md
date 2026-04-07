---
category: Backend
tags:
  - api-design
  - performance
  - database
  - interview
date: 2026-04-06T00:00:00.000Z
sections:
  - id: solution
    label: Solution
    icon: Lightbulb
    order: 1
  - id: comparison
    label: Offset vs Cursor vs Keyset
    icon: BarChart3
    order: 2
  - id: source
    label: Sources
    icon: BookOpen
    order: 99
---

# Implement Efficient Cursor-Based Pagination

Your REST API uses OFFSET/LIMIT pagination. At page 10,000 the query takes 30 seconds because it scans 10M rows. How do you implement efficient cursor-based pagination that stays fast at any offset?
