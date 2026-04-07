---
category: DevOps
tags:
  - kubernetes
  - autoscaling
  - hpa
  - interview
date: 2026-04-06T00:00:00.000Z
sections:
  - id: solution
    label: Solution
    icon: Lightbulb
    order: 1
  - id: advanced
    label: Advanced Strategies
    icon: Zap
    order: 2
  - id: custom-metrics
    label: Custom Metrics
    icon: BarChart3
    order: 3
  - id: source
    label: Sources
    icon: BookOpen
    order: 99
---
Your service already has HPA (Horizontal Pod Autoscaler) with CPU-based scaling and it works fine under normal load. But when traffic spikes suddenly — latency shoots up, requests start timing out, and users see errors. The pods ARE scaling... just not fast enough. What's wrong with CPU-only autoscaling and how do you design an autoscale strategy that actually handles traffic bursts?
