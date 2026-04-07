## Deployment Strategies

**Bad (All-or-Nothing):**
```
0% ──────────────────▶ 100%
     │
     └─ Breaks for 10%
```
Result: Immediate outage, rollback takes 10+ minutes.

**Good (Canary):**
```
0% ──▶ 5% ──▶ 10% ──▶ 50% ──▶ 100%
     │    │     │     │
     ✓    ✓     ✓     ✓  Gradual rollout, automatic rollback if errors
```

## Canary Deployment Process

```
Step 1: Deploy to 5% of traffic
→ Monitor: error rate, latency, memory
→ Wait: 5-10 minutes
→ If healthy: Continue
→ If errors: Automatic rollback

Step 2: Deploy to 10% of traffic
→ Same monitoring
→ Continue...

Step 3: Deploy to 50% of traffic
→ 30+ minutes
→ Confident now

Step 4: Deploy to 100%
→ Complete rollout
```

## Using Istio + Kubernetes

Istio VirtualService manages traffic splitting:

```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: myapp-vs
spec:
  hosts:
  - myapp.example.com
  http:
  # 5% to new version
  - match:
    - headers:
        x-version:
          exact: "v2"
    route:
    - destination:
        host: myapp-v2
  # 95% to stable version
  - route:
    - destination:
        host: myapp-v1
      weight: 95
    - destination:
        host: myapp-v2
      weight: 5
---
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: myapp-dr
spec:
  host: myapp
  trafficPolicy:
    connectionPool:
      http:
        http1MaxPendingRequests: 1
        maxRequestsPerConnection: 2
      tcp:
        maxConnections: 100
  outlierDetection:
    consecutive5xxErrors: 5
    interval: 30s
    baseEjectionTime: 30s
```

## Automated Canary with ArgoCD

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: myapp
spec:
  replicas: 10
  revisionHistoryLimit: 3
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
      - name: myapp
        image: myapp:v2
        ports:
        - containerPort: 8080
  strategy:
    canary:
      steps:
      - setWeight: 5
      - pause: {duration: 10m}
      - setWeight: 25
      - pause: {duration: 5m}
      - setWeight: 50
      - pause: {duration: 5m}
      - setWeight: 75
      - pause: {duration: 5m}
      trafficRouting:
        istio:
          virtualService:
            name: myapp-vs
      analysis:
        interval: 1m
        threshold: 5
        metrics:
        - name: error-rate
          interval: 1m
          failureLimit: 1
          query: |
            rate(requests_total{status=~"5.."}[5m])
        - name: latency
          interval: 1m
          failureLimit: 2
          query: |
            histogram_quantile(0.95, response_duration_seconds_bucket)
        - name: memory
          interval: 1m
          failureLimit: 3
          query: |
            container_memory_usage_bytes / 1024 / 1024
```

**Rollout behavior:**
1. Deploy v2, route 5% traffic
2. Pause 10 minutes
3. Monitor metrics (error rate, latency, memory)
4. If all healthy: increase to 25%
5. Repeat until 100%
6. If errors detected: automatic rollback

## Metrics to Monitor

```yaml
Monitoring Setup:
├─ Error Rate
│  └─ Alert if > 0.5% (vs baseline)
│
├─ Latency
│  └─ Alert if p95 > baseline + 20%
│
├─ Resource Usage
│  ├─ CPU: alert if > 80%
│  └─ Memory: alert if > 85%
│
├─ Custom Metrics
│  ├─ Conversion rate drop
│  ├─ Feature usage
│  └─ User complaints (logs)
│
└─ Infrastructure
   ├─ Pod restart count
   └─ Node disk space
```

## Rollback Strategy

**Automatic:**
```yaml
# If analysis fails, automatic rollback
spec:
  strategy:
    canary:
      analysis:
        metrics:
        - name: error-rate
          failureLimit: 2  # Rollback after 2 consecutive failures
```

**Manual:**
```bash
# If auto-rollback didn't trigger
kubectl rollout undo deployment/myapp

# Or with Argo Rollouts
kubectl argo rollouts undo myapp

# Verification
kubectl get rollout myapp
```

## Communication During Canary

```
Team notification:
- 9:00 AM: Canary deployment starting (v2 → 5% traffic)
- 9:15 AM: Metrics look good, increasing to 25%
- 9:25 AM: Error rate elevated, pausing at 25% for investigation
- 9:35 AM: Found issue, rolling back
- 9:45 AM: Back to v1, incident investigation begins
- Next day: Fixed bug, new canary with v2.1
```

## Cost of Canary

```
Resources needed:
├─ 2x deployments (v1 + v2) = 2x cost
├─ Load balancer routing logic
└─ Monitoring + alerts

But benefit:
├─ Catch bugs before 100% impact
├─ Reduce MTTR (mean time to recovery)
├─ Confidence in releases
└─ Happy customers (no mass outage)
```
