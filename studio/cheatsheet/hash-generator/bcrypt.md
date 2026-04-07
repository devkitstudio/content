---
title: "Bcrypt"
category: "Password Hashing"
order: 5
---

**Adaptive Hash**

- `Salt` Random 128-bit string added to defeat Rainbow Tables.
- `Rounds` Log2 of iterations. A round of 10 means 2^10 = 1024 iterations.
- `Cost` Increasing rounds increases CPU cost linearly, stopping brute force.
