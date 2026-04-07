## Infrastructure-Level Circuit Breaking

### Istio: Circuit Breaking Without Code Changes

Service meshes like Istio implement circuit breaking at the proxy (Envoy sidecar) level — no application code needed:

```yaml
# DestinationRule with circuit breaking
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: fraud-service
spec:
  host: fraud-service.default.svc.cluster.local
  trafficPolicy:
    connectionPool:
      tcp:
        maxConnections: 100          # max concurrent connections
      http:
        h2UpgradePolicy: DEFAULT
        http1MaxPendingRequests: 50  # max queued requests
        http2MaxRequests: 100        # max active requests
    outlierDetection:
      consecutive5xxErrors: 5        # trip after 5 consecutive 5xx
      interval: 10s                  # check window
      baseEjectionTime: 30s          # minimum ejection time
      maxEjectionPercent: 50         # max % of hosts to eject
      minHealthPercent: 30           # disable ejection if < 30% healthy
```

**Key advantage**: Works for ANY language/framework. Go, Python, Java, Node.js services all get circuit breaking automatically.

### Envoy (Standalone)

If you're not using a full service mesh, Envoy can still provide circuit breaking as a sidecar:

```yaml
# envoy.yaml
clusters:
  - name: fraud_service
    type: STRICT_DNS
    circuit_breakers:
      thresholds:
        - priority: DEFAULT
          max_connections: 100
          max_pending_requests: 50
          max_requests: 200
          max_retries: 3
          retry_budget:
            budget_percent: 20.0     # max 20% of active requests can be retries
            min_retry_concurrency: 3
    outlier_detection:
      consecutive_5xx: 5
      interval: 10s
      base_ejection_time: 30s
      max_ejection_percent: 50
      enforcing_consecutive_5xx: 100  # 100% enforcement
```

### Linkerd: Failure Accrual

Linkerd uses "failure accrual" instead of traditional circuit breaking:

```yaml
apiVersion: policy.linkerd.io/v1beta3
kind: HTTPRoute
metadata:
  name: fraud-service
spec:
  parentRefs:
    - name: fraud-service
      kind: Service
  rules:
    - timeouts:
        request: 3s       # per-request timeout
        backendRequest: 2s # timeout for backend specifically
      retry:
        limit: 2
        timeout: 5s
```

Linkerd automatically performs failure accrual per-endpoint and removes unhealthy pods from the load balancer.

### Application vs Infrastructure: When to Use Each

| Aspect | Application-Level | Service Mesh |
|--------|-------------------|-------------|
| **Fallback logic** | Custom per use case | Generic (503 / retry) |
| **Granularity** | Per function/endpoint | Per service/route |
| **Language support** | Language-specific lib | Any language |
| **Deployment complexity** | npm install | Full mesh setup |
| **Observability** | Manual metrics | Built-in dashboards |
| **State sharing** | In-process only | Across replicas (outlier detection) |
| **Best for** | Complex fallback logic | Uniform protection |

### Combined Approach (Recommended for Production)

Use **both** — service mesh for baseline protection, application-level for business logic:

```
                Service Mesh Layer
                (Envoy sidecar)
                    │
                    │  ← outlier detection, connection limits
                    │  ← rejects if service is fully down
                    │
              Application Layer
              (Opossum / resilience4j)
                    │
                    │  ← custom fallback logic
                    │  ← per-endpoint breakers
                    │  ← business-aware health checks
                    │
              Business Logic
```

```typescript
// Application-level breaker INSIDE the service mesh
// Mesh handles: connection limits, outlier detection, retries
// App handles: smart fallbacks, partial degradation

const fraudBreaker = new CircuitBreaker(
  (order) => fraudClient.check(order), // this call goes through Envoy
  {
    timeout: 2000,
    errorThresholdPercentage: 50,
  }
);

// Fallback: approve but flag for manual review
fraudBreaker.fallback((order) => ({
  safe: true,
  confidence: 0,
  reason: 'circuit-open-manual-review',
  flagged: true,
}));
```
