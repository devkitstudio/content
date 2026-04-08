# The Circuit Breaker Architecture

Inspired by electrical circuit breakers, this pattern prevents a localized failure in a microservice ecosystem from cascading into a global system outage.

When a downstream dependency fails or degrades heavily, the system **stops routing traffic to it** and fails fast, rather than exhausting thread pools waiting for timeouts.

### The State Machine

```text
     [CLOSED] ──failures exceed threshold──▶ [OPEN]
        ▲                                       │
        │                                  timer expires
        │                                       │
        └──── probe succeeds ◀── [HALF-OPEN] ◀──┘
```

- **CLOSED (Normal Operation):** Requests flow freely. The breaker monitors the failure rate and latency percentiles.
- **OPEN (Tripped):** The downstream service is deemed unhealthy. All incoming requests are **immediately rejected** with a fallback response or an instant error. Zero network calls are made.
- **HALF-OPEN (Probing):** After a predefined cooldown period, the breaker allows a tiny, controlled sample of requests to pass through. If they succeed, the circuit resets to `CLOSED`. If they fail, it trips back to `OPEN`.

### The Mechanics of Cascading Failure Prevention

**Without Circuit Breaker (The Cascade):**

```text
User → Order Service → Payment API → Fraud Service (DOWN)
       ↑ Thread stuck waiting for timeout
       ↑ 1,000 concurrent users = 1,000 stuck threads
       ↑ Order Service exhausts its connection pool and crashes.
```

**With Circuit Breaker (The Isolation):**

```text
User → Order Service → Circuit Breaker [OPEN] → Instant Fallback (5ms)
       ↑ Thread freed immediately
       ↑ Order Service remains 100% healthy
       ↑ Fraud Service is given time to recover without being DDOS'd.
```
