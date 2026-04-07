---
category: Architecture
tags:
  - backend
  - performance
  - database
  - interview
date: 2026-04-06T00:00:00.000Z
sections:
  - id: solution
    label: Solution
    icon: Lightbulb
    order: 1
  - id: why
    label: The Disaster
    icon: Flame
    order: 2
  - id: implementation
    label: Implement
    icon: Code2
    order: 3
  - id: db-cancellation
    label: DB Cancellation
    icon: DatabaseZap
    order: 4
  - id: frameworks
    label: Framework Patterns
    icon: Blocks
    order: 5
  - id: source
    label: Sources
    icon: BookOpen
    order: 99
---
A user clicks to download a heavy report. Tired of waiting, they angrily hit "Cancel" or "F5" to refresh. The frontend instantly drops the connection. However, the backend is completely unaware! It blindly continues to burn CPU and hammer the database for 10 more seconds to run a giant "zombie" query. If 1,000 users mash F5, the server crashes. How do we make the backend "listen" and instantly slam the brakes on the DB query?
