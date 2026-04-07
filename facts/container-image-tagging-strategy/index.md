---
category: DevOps
tags:
  - docker
  - deployment
  - ci-cd
  - experience
date: 2026-04-06T00:00:00.000Z
sections:
  - id: solution
    label: Solution
    order: 1
  - id: pitfalls
    label: Why `latest` Is Dangerous
    icon: AlertTriangle
    order: 2
  - id: source
    label: Sources
    icon: BookOpen
    order: 99
---

Your container image uses latest tag. Production mysteriously breaks even though nobody deployed. How do you implement proper container image tagging and immutable deployments?
