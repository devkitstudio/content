---
category: Backend
tags:
  - monitoring
  - devops
  - reliability
  - experience
date: 2026-04-06T00:00:00.000Z
sections:
  - id: solution
    label: Solution
    icon: Lightbulb
    order: 1
  - id: implementation
    label: Liveness vs Readiness Code
    icon: Code
    order: 2
  - id: source
    label: Sources
    icon: BookOpen
    order: 99
---

# Design Meaningful Health Checks

Your health check endpoint returns 200 OK but the service is actually broken. It can't reach the database, can't connect to external APIs, but the liveness probe says everything is fine. How do you design health checks that actually reflect service health?
