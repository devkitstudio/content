---
category: Database
tags:
  - replication
  - consistency
  - postgresql
  - interview
date: 2026-04-06T00:00:00.000Z
sections:
  - id: solution
    label: Solution
    icon: Settings
    order: 1
  - id: implementation
    label: Implementation
    icon: Code
    order: 2
  - id: source
    label: Documentation
    icon: BookOpen
    order: 99
---

Your application writes to a primary and reads from a replica. But read-after-write shows stale data because replication lag is 500ms. How do you handle replication lag?
