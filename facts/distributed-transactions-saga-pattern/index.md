---
category: Architecture
tags:
  - microservices
  - database
  - senior
  - system-design
date: 2026-04-06T00:00:00.000Z
sections:
  - id: solution
    label: Solution
    icon: Lightbulb
    order: 1
  - id: why
    label: The Alternatives
    icon: Scale
    order: 2
  - id: implementation
    label: Implementation
    icon: code-2
    order: 3
  - id: source
    label: Sources
    icon: BookOpen
    order: 99
---
In a Monolith, if an Order fails to submit, a single SQL `ROLLBACK` undoes the database. But in Microservices, the `Inventory DB` deducts the stock upon checkout, but the `Payment API` network completely fails. How do you rollback an inventory deduction on a separate database when there is no single SQL transaction spanning across the microservices?
