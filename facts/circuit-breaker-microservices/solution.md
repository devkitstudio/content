## The Circuit Breaker Pattern

Inspired by electrical circuit breakers. When a downstream service fails repeatedly, **stop calling it** and fail fast instead of hanging.

### Three States

```
     [CLOSED] ──failures exceed threshold──▶ [OPEN]
        ▲                                       │
        │                                  timer expires
        │                                       │
        └──── probe succeeds ◀── [HALF-OPEN] ◀──┘
```

- **CLOSED** (normal): Requests pass through. Count consecutive failures.
- **OPEN** (tripped): All requests **immediately rejected** with a fallback. No network call.
- **HALF-OPEN** (probing): After a cooldown, allow **one** test request. If it succeeds → CLOSED. If it fails → OPEN again.

### Why This Prevents Cascading Failure

Without Circuit Breaker:
```
User → Order Service → Payment (30s timeout) → Fraud (DOWN)
       ↑ thread stuck for 30s
       ↑ 100 users = 100 stuck threads = Order Service dies too
```

With Circuit Breaker:
```
User → Order Service → Circuit Breaker [OPEN] → instant fallback (5ms)
       ↑ thread freed immediately
       ↑ Order Service stays healthy
```

### Implementation

```typescript
class CircuitBreaker {
  private state: 'CLOSED' | 'OPEN' | 'HALF_OPEN' = 'CLOSED';
  private failures = 0;
  private lastFailure = 0;

  constructor(
    private threshold: number = 5,      // failures before tripping
    private cooldown: number = 30_000,   // ms before probing
  ) {}

  async call<T>(fn: () => Promise<T>, fallback: () => T): Promise<T> {
    if (this.state === 'OPEN') {
      if (Date.now() - this.lastFailure > this.cooldown) {
        this.state = 'HALF_OPEN'; // Try one probe
      } else {
        return fallback(); // Fail fast
      }
    }

    try {
      const result = await fn();
      this.onSuccess();
      return result;
    } catch (err) {
      this.onFailure();
      return fallback();
    }
  }

  private onSuccess() {
    this.failures = 0;
    this.state = 'CLOSED';
  }

  private onFailure() {
    this.failures++;
    this.lastFailure = Date.now();
    if (this.failures >= this.threshold) {
      this.state = 'OPEN';
    }
  }
}
```

### Usage

```typescript
const fraudBreaker = new CircuitBreaker(5, 30_000);

async function processPayment(order: Order) {
  const fraudCheck = await fraudBreaker.call(
    () => fraudService.check(order),       // normal call
    () => ({ safe: true, reason: 'skip' }) // fallback: approve and flag for manual review
  );

  if (!fraudCheck.safe) throw new Error('Fraud detected');
  return paymentGateway.charge(order);
}
```
