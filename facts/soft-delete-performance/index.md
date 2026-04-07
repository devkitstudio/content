---
category: Database
tags:
  - sql
  - patterns
  - performance
  - experience
date: 2026-04-06T00:00:00.000Z
sections:
  - id: solution
    label: Solution
    icon: Settings
    order: 1
  - id: alternatives
    label: Alternative Patterns
    icon: Layers
    order: 2
  - id: source
    label: Documentation
    icon: BookOpen
    order: 99
---

Your application uses soft deletes (deleted_at column). After 2 years, 80% of rows are "deleted" but still scanned by every query. How do you manage soft deletes efficiently?
