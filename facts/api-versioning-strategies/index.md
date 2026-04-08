---
category: Backend
tags:
  - api-design
  - versioning
  - backend
  - interview
date: 2026-04-06T00:00:00.000Z
review: true
sections:
  - id: solution
    label: Solution
    icon: Lightbulb
    order: 1
  - id: comparison
    label: Comparison & Trade-offs
    icon: bar-chart-3
    order: 2
  - id: gateway
    label: API Gateway Pattern
    icon: Network
    order: 3
  - id: source
    label: Sources
    icon: BookOpen
    order: 99
---

API schemas evolve rapidly to support new features, but legacy clients deployed in the wild (especially mobile apps) cannot be forced to update instantly. Breaking changes without a versioning strategy will trigger immediate production outages.
