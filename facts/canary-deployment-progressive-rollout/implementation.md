## Execution: Service Mesh Canary Rollout

Modern canary deployments decouple **Instance Scaling** (Kubernetes Deployments) from **Traffic Routing** (Service Mesh).

### 1. The Infrastructure Definition (Istio)

Define the stable (v1) and canary (v2) deployments independently, then use an Istio `VirtualService` to strictly govern the traffic weights.

```yaml
# virtualservice.yaml - The Traffic Controller
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: myapp-vs
spec:
  hosts:
    - myapp.example.com
  http:
    - route:
        - destination:
            host: myapp
            subset: v1
          weight: 95 # 95% of traffic remains on the stable version
        - destination:
            host: myapp
            subset: v2
          weight: 5 # 5% blast radius for the canary version
      timeout: 30s
```

### 2. The Agnostic Observability Contract (Prometheus)

Do not hardcode absolute limits (like `> 500MB` memory). An effective Canary monitor compares the new version's telemetry directly against the stable version's baseline in real-time.

```yaml
# prometheus-rules.yaml
apiVersion: [monitoring.coreos.com/v1](https://monitoring.coreos.com/v1)
kind: PrometheusRule
metadata:
  name: canary-alerts
spec:
  groups:
  - name: canary-evaluation
    interval: 1m
    rules:
    # 1. Relative Error Rate Anomaly
    - alert: CanaryErrorRateSpike
      expr: |
        (rate(http_requests_total{version="v2", status=~"5.."}[5m]) / rate(http_requests_total{version="v2"}[5m]))
        >
        (rate(http_requests_total{version="v1", status=~"5.."}[5m]) / rate(http_requests_total{version="v1"}[5m])) * 1.5
      for: 2m
      annotations:
        summary: "Canary (v2) error rate is 50% higher than Stable (v1)"

    # 2. Relative Latency Degradation
    - alert: CanaryLatencyHigh
      expr: |
        histogram_quantile(0.95, rate(http_request_duration_seconds_bucket{version="v2"}[5m]))
        >
        histogram_quantile(0.95, rate(http_request_duration_seconds_bucket{version="v1"}[5m])) * 1.2
      for: 2m
      annotations:
        summary: "Canary (v2) p95 latency is 20% slower than Stable (v1)"

    # 3. Relative Memory Leak Detection
    - alert: CanaryMemoryAnomaly
      expr: |
        avg(container_memory_usage_bytes{version="v2"})
        >
        avg(container_memory_usage_bytes{version="v1"}) * 1.3
      for: 10m
      annotations:
        summary: "Canary (v2) is consuming 30% more memory than Stable (v1). Potential leak."
```

### 3. The Rollback Logic

If any of the Prometheus alerts trigger, the system must immediately patch the `VirtualService` to route 100% of traffic back to `v1`.

_Note: While the bash script below demonstrates the mechanical logic, production environments should use GitOps controllers (like **Flagger** or **Argo Rollouts**) to automate this evaluation and rollback loop natively within the cluster._

```bash
#!/bin/bash
# Concept: Automated Rollback Circuit Breaker

# 1. Query Prometheus for Active Canary Alerts
ALERTS=$(curl -s http://prometheus:9090/api/v1/alerts | jq '.data.alerts | length')

if [ "$ALERTS" -gt 0 ]; then
  echo "CRITICAL: Canary degradation detected. Executing emergency rollback..."

  # 2. Instantly kill canary traffic at the mesh level
  kubectl patch vs myapp-vs --type merge -p '
  spec:
    http:
    - route:
      - destination: {host: myapp, subset: v1}
        weight: 100
      - destination: {host: myapp, subset: v2}
        weight: 0
  '

  echo "Traffic strictly isolated to v1. Paging on-call engineer."
fi
```
