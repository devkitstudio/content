## Production Implementations

Do not write your own state machine in production. Use battle-tested libraries or infrastructure configurations.

### 1. Node.js (Opossum)

Wrapping a fraud check service with a custom fallback response.

```typescript
import CircuitBreaker from "opossum";

const breaker = new CircuitBreaker(fraudService.check, {
  timeout: 3000, // 3s timeout per call
  errorThresholdPercentage: 50, // trip if 50% of requests fail
  resetTimeout: 30000, // wait 30s before trying HALF-OPEN
});

// Fallback: If Fraud service is down, approve the transaction but flag it
breaker.fallback(() => ({
  safe: true,
  confidence: 0,
  reason: "circuit-open-manual-review",
}));

const result = await breaker.fire(orderData);
```

### 2. Java (Resilience4j)

The JVM standard, configuring strict sliding windows and half-open probes.

```java
CircuitBreakerConfig config = CircuitBreakerConfig.custom()
    .failureRateThreshold(50)
    .waitDurationInOpenState(Duration.ofSeconds(30))
    .permittedNumberOfCallsInHalfOpenState(3)  // Allow 3 probes, not just 1
    .slidingWindowSize(10)                     // Calculate over last 10 calls
    .build();

CircuitBreaker breaker = CircuitBreaker.of("fraudService", config);
Supplier<FraudResult> decorated = CircuitBreaker.decorateSupplier(breaker, () -> fraudService.check(order));
Try<FraudResult> result = Try.ofSupplier(decorated).recover(Exception.class, e -> FraudResult.skip());
```

### 3. Infrastructure (Istio DestinationRule)

Applying a circuit breaker via Service Mesh without touching application code.

```yaml
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: fraud-service
spec:
  host: fraud-service.default.svc.cluster.local
  trafficPolicy:
    outlierDetection:
      consecutive5xxErrors: 5 # Trip after 5 consecutive 5xx responses
      interval: 10s # Evaluation window
      baseEjectionTime: 30s # Time to keep the circuit OPEN
      maxEjectionPercent: 50 # Don't eject more than 50% of pods
```
