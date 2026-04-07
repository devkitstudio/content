---
category: Architecture
tags:
  - redis
  - leaderboard
  - system-design
  - interview
date: 2026-04-06T00:00:00.000Z
sections:
  - id: solution
    label: Solution
    icon: Lightbulb
    order: 1
  - id: implementation
    label: Implementation
    icon: Code
    order: 2
  - id: source
    label: Sources
    icon: BookOpen
    order: 99
---

# Real-Time Leaderboard with Redis Sorted Sets

You need to build a leaderboard that updates in real-time for 1M users. SQL ORDER BY with LIMIT is too slow. How do you design a real-time leaderboard with Redis Sorted Sets?
