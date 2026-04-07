---
category: Backend
tags:
  - microservices
  - messaging
  - consistency
  - senior
date: 2026-04-06T00:00:00.000Z
sections:
  - id: solution
    label: Solution
    order: 1
  - id: implementation
    label: Prisma + Outbox
    icon: Code
    order: 2
  - id: source
    label: Sources
    icon: BookOpen
    order: 99
---

Your service publishes events to Kafka but sometimes the database write succeeds and the event publish fails. How does the Transactional Outbox pattern solve this?
