## Production-Ready Libraries

### Node.js: Opossum

The most popular Circuit Breaker library for Node.js:

```typescript
import CircuitBreaker from 'opossum';

const breaker = new CircuitBreaker(fraudService.check, {
  timeout: 3000,          // 3s timeout per call
  errorThresholdPercentage: 50, // trip if 50% of requests fail
  resetTimeout: 30000,    // try again after 30s
  volumeThreshold: 10,    // minimum 10 requests before calculating error %
  rollingCountTimeout: 10000,  // 10s rolling window
});

// Events for monitoring
breaker.on('open',     () => metrics.increment('circuit.open'));
breaker.on('halfOpen', () => metrics.increment('circuit.halfOpen'));
breaker.on('close',    () => metrics.increment('circuit.close'));
breaker.on('fallback', () => metrics.increment('circuit.fallback'));
breaker.on('timeout',  () => metrics.increment('circuit.timeout'));

// Fallback response
breaker.fallback(() => ({ safe: true, source: 'fallback' }));

// Usage
const result = await breaker.fire(orderData);
```

### Java: Resilience4j

The de-facto standard for JVM applications (successor to Netflix Hystrix):

```java
CircuitBreakerConfig config = CircuitBreakerConfig.custom()
    .failureRateThreshold(50)           // 50% failure rate to trip
    .slowCallRateThreshold(80)          // 80% of calls exceeding slowCallDurationThreshold
    .slowCallDurationThreshold(Duration.ofSeconds(2))
    .waitDurationInOpenState(Duration.ofSeconds(30))
    .permittedNumberOfCallsInHalfOpenState(3)  // 3 probe calls, not just 1
    .slidingWindowType(SlidingWindowType.COUNT_BASED)
    .slidingWindowSize(10)
    .minimumNumberOfCalls(5)
    .build();

CircuitBreaker breaker = CircuitBreaker.of("fraudService", config);

// Decorate and call
Supplier<FraudResult> decorated = CircuitBreaker
    .decorateSupplier(breaker, () -> fraudService.check(order));

Try<FraudResult> result = Try.ofSupplier(decorated)
    .recover(CallNotPermittedException.class, e -> FraudResult.skip());
```

### Python: pybreaker

```python
import pybreaker

breaker = pybreaker.CircuitBreaker(
    fail_max=5,
    reset_timeout=30,
    exclude=[ValueError],  # don't count validation errors
)

@breaker
def check_fraud(order):
    return fraud_service.check(order)

try:
    result = check_fraud(order)
except pybreaker.CircuitBreakerError:
    result = FraudResult(safe=True, source="fallback")
```

### Go: sony/gobreaker

```go
cb := gobreaker.NewCircuitBreaker(gobreaker.Settings{
    Name:        "fraud-service",
    MaxRequests: 3,                       // half-open probes
    Interval:    10 * time.Second,        // rolling window
    Timeout:     30 * time.Second,        // open → half-open
    ReadyToTrip: func(counts gobreaker.Counts) bool {
        return counts.ConsecutiveFailures > 5
    },
    OnStateChange: func(name string, from, to gobreaker.State) {
        log.Printf("Circuit %s: %s → %s", name, from, to)
    },
})

result, err := cb.Execute(func() (interface{}, error) {
    return fraudClient.Check(ctx, order)
})
```

## Advanced Configuration

### Metrics-Based Tripping (Not Just Error Count)

Simple failure counting misses slow degradation. Use percentile-based triggers:

```typescript
// Opossum with custom health check
const breaker = new CircuitBreaker(fn, {
  errorThresholdPercentage: 50,    // error rate
  volumeThreshold: 20,             // min sample size
  rollingCountTimeout: 60000,      // 60s window

  // Custom: also trip on latency degradation
  errorFilter: (err) => {
    // Don't count 4xx as failures (client errors)
    if (err.status >= 400 && err.status < 500) return false;
    return true; // count 5xx and network errors
  },
});

// Separate latency-based breaker
const latencyBreaker = new CircuitBreaker(fn, {
  timeout: 2000,  // if p99 exceeds 2s, timeout triggers trip
  errorThresholdPercentage: 30, // lower threshold for slowness
});
```

### Per-Endpoint Circuit Breakers

Don't use one breaker per service — use one per endpoint:

```typescript
// BAD: one breaker for entire payment service
const paymentBreaker = new CircuitBreaker(paymentService);

// GOOD: separate breakers per endpoint
const chargeBreaker = new CircuitBreaker(paymentService.charge);
const refundBreaker = new CircuitBreaker(paymentService.refund);
const balanceBreaker = new CircuitBreaker(paymentService.getBalance);

// Now a failing refund endpoint doesn't block charges
```

### Circuit Breaker Registry Pattern

```typescript
class BreakerRegistry {
  private breakers = new Map<string, CircuitBreaker>();

  get(name: string, fn: Function, options?: object): CircuitBreaker {
    if (!this.breakers.has(name)) {
      this.breakers.set(name, new CircuitBreaker(fn, {
        ...defaultOptions,
        ...options,
      }));
    }
    return this.breakers.get(name)!;
  }

  // Health dashboard: report all breaker states
  getHealth(): Record<string, { state: string; stats: object }> {
    const health: Record<string, any> = {};
    for (const [name, breaker] of this.breakers) {
      health[name] = {
        state: breaker.status.name,
        stats: breaker.stats,
      };
    }
    return health;
  }
}

// Expose via health endpoint
app.get('/health/circuits', (req, res) => {
  res.json(registry.getHealth());
});
```
