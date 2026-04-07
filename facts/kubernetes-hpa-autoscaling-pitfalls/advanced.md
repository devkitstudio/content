## Beyond HPA: Strategies for Extreme Bursts

### KEDA (Event-Driven Autoscaling)

When traffic comes from message queues, cron jobs, or external events, HPA can't see it. KEDA scales based on **queue depth**, Kafka lag, Prometheus queries, and 50+ other sources.

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: api-scaler
spec:
  scaleTargetRef:
    name: api
  minReplicaCount: 3
  maxReplicaCount: 100
  triggers:
    # Scale based on pending messages in RabbitMQ
    - type: rabbitmq
      metadata:
        queueName: orders
        queueLength: "50"    # scale when >50 messages pending

    # Scale based on Prometheus latency metric
    - type: prometheus
      metadata:
        serverAddress: http://prometheus:9090
        metricName: http_request_duration_p99
        query: |
          histogram_quantile(0.99, rate(http_request_duration_seconds_bucket{service="api"}[1m]))
        threshold: "0.5"     # scale when P99 > 500ms
```

### Predictive Autoscaling (Schedule-Based)

If you know when traffic spikes happen (e.g., 9 AM business hours, Black Friday), pre-scale BEFORE the spike arrives.

```yaml
# CronJob that scales up before known peak hours
apiVersion: batch/v1
kind: CronJob
metadata:
  name: pre-scale-morning
spec:
  schedule: "45 8 * * 1-5"    # 8:45 AM, Mon-Fri
  jobTemplate:
    spec:
      template:
        spec:
          containers:
            - name: scaler
              image: bitnami/kubectl
              command:
                - kubectl
                - patch
                - hpa/api-hpa
                - --patch
                - '{"spec":{"minReplicas":10}}'
          restartPolicy: OnFailure

---
# Scale back down at night
apiVersion: batch/v1
kind: CronJob
metadata:
  name: scale-down-night
spec:
  schedule: "0 22 * * *"     # 10 PM daily
  jobTemplate:
    spec:
      template:
        spec:
          containers:
            - name: scaler
              image: bitnami/kubectl
              command:
                - kubectl
                - patch
                - hpa/api-hpa
                - --patch
                - '{"spec":{"minReplicas":3}}'
          restartPolicy: OnFailure
```

### Optimize Pod Startup Time

Scaling is useless if pods take 2 minutes to become ready. Target: **< 10 seconds from schedule to serving traffic**.

| Bottleneck | Fix |
|-----------|-----|
| Image pull (30-60s) | Use `imagePullPolicy: IfNotPresent` + pre-pull DaemonSet |
| Large image (1GB+) | Multi-stage build, distroless base, < 100MB |
| Slow app warmup | Lazy-load caches, background warmup, startup probes |
| JVM cold start | GraalVM native image, or CRaC (Checkpoint/Restore) |
| DB connection pool | Start with min pool, expand lazily |

```yaml
# Startup probe: give the app time to warm up without counting as "unhealthy"
startupProbe:
  httpGet:
    path: /healthz
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 2
  failureThreshold: 30    # 5 + (2 * 30) = 65s max startup time

# Readiness probe: only route traffic when truly ready
readinessProbe:
  httpGet:
    path: /ready
    port: 8080
  periodSeconds: 5
  failureThreshold: 2
```

## The Complete Autoscaling Checklist

- Use **multiple metrics** (CPU + RPS + latency P99)
- Set CPU target to **50-60%**, not 80%
- **Scale up fast** (stabilization = 0), **scale down slow** (stabilization = 300s)
- Keep `minReplicas` at baseline + buffer
- Optimize pod startup to < 10s
- Pre-scale for **predictable** traffic patterns
- Add **circuit breakers** to protect during the scaling gap
- Monitor with `kubectl get hpa -w` and set alerts on scaling events
