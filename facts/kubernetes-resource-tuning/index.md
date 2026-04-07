---
category: DevOps
tags:
  - kubernetes
  - resources
  - oom
  - experience
date: 2026-04-06T00:00:00.000Z
sections:
  - id: solution
    label: Solution
    icon: Zap
    order: 1
  - id: debugging
    label: Debugging & Right-Sizing
    icon: Bug
    order: 2
  - id: source
    label: Sources
    icon: BookOpen
    order: 99
---

# Kubernetes Resource Tuning

Your Kubernetes pod gets OOMKilled randomly. Memory limits are set to 512Mi but the app sometimes spikes to 600Mi. How do you tune resource requests and limits?
