## Execution: Tunable Circuit Breakers

Do not write your own state machine in production. Use battle-tested libraries or infrastructure proxies.

**Architectural Mandate:** Never hardcode breaker thresholds inside the application code. These parameters must be injectable via environment variables or a remote configuration center (like Consul or AWS Parameter Store) to allow Ops teams to tune the system during an incident without triggering a full CI/CD redeployment.

### 1. Application-Level: Node.js (Opossum)

Wrapping a downstream client with dynamic configurations and a graceful fallback.

```typescript
import CircuitBreaker from "opossum";

// INJECTED CONFIGURATION: Values must come from Env/Config, not hardcoded.
const breakerConfig = {
  timeout: process.env.FRAUD_CB_TIMEOUT_MS || 3000,
  errorThresholdPercentage: process.env.FRAUD_CB_ERROR_RATE || 50,
  resetTimeout: process.env.FRAUD_CB_COOLDOWN_MS || 30000,
};

const fraudBreaker = new CircuitBreaker(fraudService.check, breakerConfig);

// Fallback: If Fraud service is down, approve the transaction but flag it for manual audit
fraudBreaker.fallback(() => ({
  safe: true,
  confidence: 0,
  reason: "circuit-open-manual-review",
}));

const result = await fraudBreaker.fire(orderData);
```

### 2. Application-Level: Java (Resilience4j)

The JVM standard. In modern frameworks like Spring Boot, do not use the builder pattern in code. Define the limits entirely in your configuration files (`application.yml`).

```yaml
# application.yml
resilience4j.circuitbreaker:
  instances:
    fraudService:
      registerHealthIndicator: true
      slidingWindowSize: 10
      permittedNumberOfCallsInHalfOpenState: 3
      slidingWindowType: COUNT_BASED
      minimumNumberOfCalls: 5
      waitDurationInOpenState: 30s
      failureRateThreshold: 50
```

```java
// Application Code: Completely agnostic of thresholds
@CircuitBreaker(name = "fraudService", fallbackMethod = "fraudFallback")
public FraudResult checkFraud(Order order) {
    return fraudService.check(order);
}

public FraudResult fraudFallback(Order order, Exception e) {
    return FraudResult.skipAndFlag();
}
```

### 3. Infrastructure-Level: Service Mesh (Istio)

Applying a circuit breaker via Envoy proxy requires zero application code changes.

**Architectural Note:** In production Kubernetes environments, do not apply static YAML files. These values must be templated (e.g., using **Helm** or Kustomize) so they can be tuned per environment without altering the core manifest.

```yaml
# Helm Template (destination-rule.yaml)
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: fraud-service
spec:
  host: fraud-service.default.svc.cluster.local
  trafficPolicy:
    outlierDetection:
      # Thresholds are injected via Helm values (values.yaml)
      consecutive5xxErrors: { { .Values.circuitBreaker.maxErrors } }
      interval: { { .Values.circuitBreaker.evaluationInterval } }
      baseEjectionTime: { { .Values.circuitBreaker.baseEjectionTime } }
      maxEjectionPercent: { { .Values.circuitBreaker.maxEjectionPercent } }
```
