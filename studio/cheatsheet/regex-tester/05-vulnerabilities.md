---
title: "ReDoS & Catastrophic Backtracking"
category: "Security & Vulnerabilities"
order: 5
---

**Regular Expression Denial of Service (ReDoS)** occurs when a regex engine takes exponential time to resolve a complex pattern (known as *Catastrophic Backtracking*).

For example, a pattern like `(a+)+b` tested against a long string of `a`s without a `b` will cause the engine to exponentially backtrack through every possible combination.

> **Note for v0.x.x Users:** 
> This tool currently evaluates regex synchronously on the main thread. Testing a catastrophically vulnerable regex will **freeze or crash your browser tab**.
> 
> *Full Web Worker isolation with execution timeout safeguards will be released in v1.0.0!*
