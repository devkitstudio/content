## Complete Canary Implementation

```yaml
# deployment-v1.yaml - Stable version
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-v1
spec:
  replicas: 5
  selector:
    matchLabels:
      app: myapp
      version: v1
  template:
    metadata:
      labels:
        app: myapp
        version: v1
    spec:
      containers:
      - name: myapp
        image: myapp:v1.0.0
        ports:
        - containerPort: 8080
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /ready
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 5
---
# deployment-v2.yaml - Canary version
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-v2
spec:
  replicas: 1  # Start with 1 replica
  selector:
    matchLabels:
      app: myapp
      version: v2
  template:
    metadata:
      labels:
        app: myapp
        version: v2
    spec:
      containers:
      - name: myapp
        image: myapp:v2.0.0
        ports:
        - containerPort: 8080
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /ready
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 5
---
# service.yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp
spec:
  selector:
    app: myapp
  ports:
  - port: 80
    targetPort: 8080
  type: ClusterIP
---
# virtualservice.yaml - 95% v1, 5% v2
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: myapp-vs
spec:
  hosts:
  - myapp.example.com
  - myapp
  http:
  - match:
    - uri:
        prefix: /admin
    route:
    - destination:
        host: myapp
        port:
          number: 80
        subset: v1
  - route:
    - destination:
        host: myapp
        port:
          number: 80
        subset: v1
      weight: 95
    - destination:
        host: myapp
        port:
          number: 80
        subset: v2
      weight: 5
    timeout: 30s
---
# destinationrule.yaml
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: myapp
spec:
  host: myapp
  trafficPolicy:
    connectionPool:
      http:
        http1MaxPendingRequests: 100
        http2MaxRequests: 1000
        maxRequestsPerConnection: 2
    outlierDetection:
      consecutive5xxErrors: 3
      interval: 30s
      baseEjectionTime: 30s
      maxEjectionPercent: 50
  subsets:
  - name: v1
    labels:
      version: v1
  - name: v2
    labels:
      version: v2
```

## Manual Canary Steps

```bash
# Step 1: Deploy v2 (0% traffic initially)
kubectl apply -f deployment-v2.yaml

# Step 2: Wait for v2 to be ready
kubectl wait --for=condition=ready pod \
  -l version=v2 --timeout=300s

# Step 3: Route 5% traffic to v2
kubectl patch vs myapp-vs --type merge -p '
spec:
  http:
  - route:
    - destination: {host: myapp, subset: v1}
      weight: 95
    - destination: {host: myapp, subset: v2}
      weight: 5
'

# Step 4: Monitor (5-10 minutes)
kubectl logs -f deployment/myapp-v2 --all-containers
# Check metrics in Prometheus/Grafana

# Step 5: Increase to 25%
kubectl patch vs myapp-vs --type merge -p '
spec:
  http:
  - route:
    - destination: {host: myapp, subset: v1}
      weight: 75
    - destination: {host: myapp, subset: v2}
      weight: 25
'

# Step 6: Continue to 50%, 75%, 100%
# ... repeat above

# Step 7: Scale down v1
kubectl scale deployment myapp-v1 --replicas=0

# Step 8: Remove v1 deployment after retention
kubectl delete deployment myapp-v1
```

## Prometheus Queries for Monitoring

```yaml
# prometheus-rules.yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: canary-alerts
spec:
  groups:
  - name: canary
    interval: 1m
    rules:
    # Error rate alert
    - alert: CanaryErrorRateHigh
      expr: |
        (rate(request_duration_seconds_count{status=~"5.."}[5m]) >
         rate(request_duration_seconds_count[5m]) * 0.01)
        and on(job) metric{version="v2"}
      for: 2m
      annotations:
        summary: "Canary v2 error rate > 1%"

    # Latency alert
    - alert: CanaryLatencyHigh
      expr: |
        histogram_quantile(0.95, rate(request_duration_seconds_bucket{version="v2"}[5m])) >
        histogram_quantile(0.95, rate(request_duration_seconds_bucket{version="v1"}[5m])) * 1.2
      for: 2m
      annotations:
        summary: "Canary v2 latency 20% higher than v1"

    # Memory alert
    - alert: CanaryMemoryUsageHigh
      expr: |
        container_memory_usage_bytes{pod=~"myapp-v2.*"} / 1024 / 1024 > 500
      for: 2m
      annotations:
        summary: "Canary v2 using > 500MB memory"
```

## Automatic Rollback Script

```bash
#!/bin/bash
# rollback-canary.sh

NAMESPACE="production"
STABLE_VERSION="v1"
CANARY_VERSION="v2"

# Check error rate
ERROR_RATE=$(kubectl exec -n $NAMESPACE \
  deployment/myapp-$CANARY_VERSION \
  -- curl -s http://localhost:8000/metrics | \
  grep 'request_duration_seconds_count{status="500"}'  | \
  awk '{print $2}')

if (( $(echo "$ERROR_RATE > 0.01" | bc -l) )); then
  echo "Error rate too high: $ERROR_RATE"
  echo "Rolling back canary..."

  kubectl patch vs myapp-vs --type merge -p '
  spec:
    http:
    - route:
      - destination: {host: myapp, subset: v1}
        weight: 100
      - destination: {host: myapp, subset: v2}
        weight: 0
  '

  echo "Rolled back to v1"
  # Notify team
  curl -X POST -H 'Content-type: application/json' \
    --data '{"text":"Canary rollback executed"}' \
    $SLACK_WEBHOOK_URL
fi
```
