---
category: Database
tags:
  - sql
  - performance
  - experience
  - optimization
date: 2026-04-06T00:00:00.000Z
sections:
  - id: solution
    label: The Anti-Patterns
    icon: Lightbulb
    order: 1
  - id: checklist
    label: Index Checklist
    icon: CheckCircle
    order: 2
  - id: source
    label: Sources
    icon: BookOpen
    order: 99
---
You added an index on `users.email` to speed up login queries. It worked. So you added indexes on every column "just in case." Now your INSERT throughput dropped 60%, your disk usage tripled, and the query planner is ignoring half your indexes anyway. When do indexes actually hurt performance, and what are the most common indexing mistakes?
