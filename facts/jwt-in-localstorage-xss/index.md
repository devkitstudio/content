---
category: Security
tags:
  - jwt
  - auth
  - security
  - interview
  - frontend
date: 2026-04-06T00:00:00.000Z
sections:
  - id: xss_attack
    label: XSS Attack
    icon: Flame
    order: 1
  - id: httponly_cookies
    label: The Solution
    icon: Cookie
    order: 2
  - id: csrf
    label: CSRF Trade-off
    icon: ShieldAlert
    order: 3
  - id: token-strategies
    label: Token Strategies
    icon: Key
    order: 4
  - id: source
    label: Sources
    icon: BookOpen
    order: 99
---
Every tutorial tells you to build your React app, get the JWT token from the login API, and do `localStorage.setItem('token', jwt)`. 
However, Senior Security Engineers will instantly fail your PR if you do this. Why is putting JWTs in LocalStorage considered a critical security vulnerability, and how should you actually store them?
