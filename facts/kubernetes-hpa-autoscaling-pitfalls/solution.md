## Why CPU-Only HPA Fails on Traffic Spikes

The core problem is **reaction lag**. CPU-based HPA is a chain of slow steps:

```
Traffic spike arrives
  → Requests queue up (latency rising)
    → CPU eventually climbs past threshold
      → HPA detects high CPU (scrape interval: 15-30s)
        → HPA decides to scale (stabilization window: 30-60s)
          → New pod scheduled (scheduler: 1-5s)
            → Container image pulled (pull: 5-60s)
              → App starts and warms up (startup: 10-30s)
                → Readiness probe passes (probe: 10-30s)
                  → Pod receives traffic
```

**Total delay: 1-3 minutes.** By then, users have already seen errors.

### The Root Cause: CPU Is a Lagging Indicator

- CPU tells you the system is **already overloaded**
- By the time CPU hits 80%, requests are already queuing and timing out
- You need **leading indicators**: metrics that predict overload before it happens

## The Fix: Multi-Signal Autoscaling

### 1. Scale on Request Rate (RPS), Not Just CPU

RPS is a **leading indicator** — it rises immediately when traffic spikes, before CPU even moves.

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: api-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: api
  minReplicas: 3
  maxReplicas: 50
  metrics:
    # Signal 1: CPU (baseline)
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 60   # scale earlier, not at 80%

    # Signal 2: RPS per pod (leading indicator)
    - type: Pods
      pods:
        metric:
          name: http_requests_per_second
        target:
          type: AverageValue
          averageValue: "100"      # scale when any pod exceeds 100 RPS
```

HPA picks the metric that demands the **most replicas** — so RPS can trigger scaling before CPU even rises.

### 2. Lower the CPU Threshold

Most teams set CPU target at 70-80%. That's too late.

```yaml
# Bad: scales when already overloaded
averageUtilization: 80

# Better: scales while there's still headroom
averageUtilization: 50
```

The cost of a few extra pods is almost always less than the cost of user-facing errors.

### 3. Set Aggressive Scale-Up, Conservative Scale-Down

```yaml
spec:
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 0    # scale up IMMEDIATELY
      policies:
        - type: Percent
          value: 100                   # double pods in one step
          periodSeconds: 15
        - type: Pods
          value: 10                    # or add 10 pods at once
          periodSeconds: 15
      selectPolicy: Max                # pick whichever adds more

    scaleDown:
      stabilizationWindowSeconds: 300  # wait 5 min before scaling down
      policies:
        - type: Percent
          value: 10                    # remove only 10% at a time
          periodSeconds: 60
```

**Why asymmetric?** Scaling up too slow = user errors. Scaling down too fast = traffic returns and you get errors again.

### 4. Keep a Warm Buffer with minReplicas

```yaml
# Bad: starts from 1 pod, cold start on every spike
minReplicas: 1

# Better: always keep enough pods to handle baseline + small bursts
minReplicas: 3
```

Calculate: `minReplicas = (baseline_RPS / RPS_per_pod) * 1.5`
