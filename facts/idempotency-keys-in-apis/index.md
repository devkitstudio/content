---
category: Backend
tags:
  - architecture
  - senior
  - api
  - stripe
date: 2026-04-06T00:00:00.000Z
sections:
  - id: solution
    label: Solution
    icon: ShieldCheck
    order: 1
  - id: implementation
    label: Implementation
    icon: Code2
    order: 2
  - id: patterns
    label: Patterns & Storage
    icon: Database
    order: 3
  - id: source
    label: Sources
    icon: BookOpen
    order: 99
---
A mobile user on a flaky 3G connection hits the "Pay $100" button twice because nothing loaded. Your API receives two identical HTTP POST requests at the exact same millisecond. How do you prevent charging their credit card $200 while maintaining a graceful user experience?
