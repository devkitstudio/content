---
category: Database
tags:
  - ecommerce
  - concurrency
  - locking
  - interview
date: 2026-04-06T00:00:00.000Z
sections:
  - id: solution
    label: Atomic Update
    icon: Zap
    order: 1
  - id: optimistic
    label: Optimistic Lock
    icon: FastForward
    order: 2
  - id: pessimistic
    label: Pessimistic Lock
    icon: Lock
    order: 3
  - id: distributed
    label: Distributed Solutions
    icon: Globe
    order: 4
  - id: source
    label: Sources
    icon: BookOpen
    order: 99
---
During a Flash Sale, User A and User B both view a product page. The database says `stock = 1`. They click "Buy" at the exact same millisecond.

User A reads stock: 1.
User B reads stock: 1.
User A bypasses the `if (stock > 0)` check and updates stock to 0. 
User B also bypasses the `if (stock > 0)` check (because they read 1 earlier) and updates stock to 0.

You just sold 2 iPhones when you only had 1. How do you prevent this classic Database Race Condition (Overselling)?
