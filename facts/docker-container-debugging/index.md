---
category: DevOps
tags:
  - docker
  - debugging
  - experience
  - deployment
date: 2026-04-06T00:00:00.000Z
sections:
  - id: solution
    label: Debug Playbook
    icon: Lightbulb
    order: 1
  - id: war-stories
    label: War Stories
    icon: Flame
    order: 2
  - id: source
    label: Sources
    icon: BookOpen
    order: 99
---
It's 2 AM. You deploy a new Docker image to production. `docker ps` shows the container restarting every 10 seconds. `docker logs` shows nothing useful — just "Killed." You run `docker inspect` and see `OOMKilled: true`. What is your systematic approach to debug a container that keeps crashing, and what are the most common causes beyond OOM?
