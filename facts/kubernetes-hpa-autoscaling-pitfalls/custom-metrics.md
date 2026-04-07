## Setting Up Custom Metrics for HPA

### Prometheus + Prometheus Adapter Pipeline

HPA needs a metrics API to read from. The standard pipeline:

```
App → exports metrics → Prometheus scrapes → Prometheus Adapter → HPA reads
```

### Step 1: Expose Metrics from Your App

```typescript
// Express/Fastify: expose Prometheus metrics
import { Registry, Counter, Histogram } from 'prom-client';

const register = new Registry();

const httpRequests = new Counter({
  name: 'http_requests_total',
  help: 'Total HTTP requests',
  labelNames: ['method', 'path', 'status'],
  registers: [register],
});

const httpDuration = new Histogram({
  name: 'http_request_duration_seconds',
  help: 'HTTP request duration',
  labelNames: ['method', 'path'],
  buckets: [0.01, 0.05, 0.1, 0.5, 1, 5],
  registers: [register],
});

// Middleware
app.use((req, res, next) => {
  const end = httpDuration.startTimer({ method: req.method, path: req.path });
  res.on('finish', () => {
    httpRequests.inc({ method: req.method, path: req.path, status: res.statusCode });
    end();
  });
  next();
});

// Metrics endpoint (Prometheus scrapes this)
app.get('/metrics', async (req, res) => {
  res.set('Content-Type', register.contentType);
  res.end(await register.metrics());
});
```

### Step 2: Configure Prometheus Adapter

```yaml
# prometheus-adapter-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: adapter-config
  namespace: monitoring
data:
  config.yaml: |
    rules:
      # RPS per pod
      - seriesQuery: 'http_requests_total{namespace!="",pod!=""}'
        resources:
          overrides:
            namespace: {resource: "namespace"}
            pod: {resource: "pod"}
        name:
          matches: "^(.*)_total$"
          as: "${1}_per_second"
        metricsQuery: 'sum(rate(<<.Series>>{<<.LabelMatchers>>}[2m])) by (<<.GroupBy>>)'

      # P99 latency per pod
      - seriesQuery: 'http_request_duration_seconds_bucket{namespace!="",pod!=""}'
        resources:
          overrides:
            namespace: {resource: "namespace"}
            pod: {resource: "pod"}
        name:
          as: "http_request_p99_seconds"
        metricsQuery: 'histogram_quantile(0.99, sum(rate(<<.Series>>{<<.LabelMatchers>>}[2m])) by (le, <<.GroupBy>>))'

      # Queue depth (for async workers)
      - seriesQuery: 'job_queue_length{namespace!="",pod!=""}'
        resources:
          overrides:
            namespace: {resource: "namespace"}
            pod: {resource: "pod"}
        name:
          as: "job_queue_depth"
        metricsQuery: 'avg(<<.Series>>{<<.LabelMatchers>>}) by (<<.GroupBy>>)'
```

### Step 3: HPA with Custom Metrics

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
  maxReplicas: 100
  metrics:
    # CPU baseline
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 50

    # Custom: RPS per pod
    - type: Pods
      pods:
        metric:
          name: http_requests_per_second
        target:
          type: AverageValue
          averageValue: "100"

    # Custom: P99 latency threshold
    - type: Pods
      pods:
        metric:
          name: http_request_p99_seconds
        target:
          type: AverageValue
          averageValue: "0.5"  # scale up if p99 > 500ms

    # External: queue depth from RabbitMQ/SQS
    - type: External
      external:
        metric:
          name: queue_messages_ready
          selector:
            matchLabels:
              queue: "order-processing"
        target:
          type: AverageValue
          averageValue: "10"  # target 10 msgs per pod
```

### Vertical Pod Autoscaler (VPA) — Complement to HPA

HPA scales horizontally (more pods). VPA scales vertically (bigger pods):

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: api-vpa
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: api
  updatePolicy:
    updateMode: "Auto"  # or "Off" to only get recommendations
  resourcePolicy:
    containerPolicies:
      - containerName: api
        minAllowed:
          cpu: 100m
          memory: 128Mi
        maxAllowed:
          cpu: 2
          memory: 2Gi
```

**Warning**: Don't use VPA and HPA on CPU/memory simultaneously — they'll fight. Use HPA for scaling on custom metrics + VPA for right-sizing resource requests.

### Cost Optimization: Scale-to-Zero with KEDA

For workloads with idle periods, KEDA can scale to zero pods:

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: worker-scaler
spec:
  scaleTargetRef:
    name: order-worker
  minReplicaCount: 0    # scale to ZERO when idle
  maxReplicaCount: 50
  cooldownPeriod: 300   # wait 5 min before scaling to zero
  triggers:
    - type: rabbitmq
      metadata:
        queueName: orders
        host: amqp://rabbitmq.default.svc:5672
        queueLength: "5"  # 1 pod per 5 messages

    - type: cron
      metadata:
        timezone: Asia/Ho_Chi_Minh
        start: 0 8 * * 1-5    # scale up at 8am weekdays
        end: 0 20 * * 1-5     # scale down at 8pm
        desiredReplicas: "3"   # minimum during business hours
```

### Complete Autoscaling Strategy (Interview Answer)

```
1. HPA with multi-signal:
   - CPU (50% target, not 80%)
   - RPS per pod (leading indicator)
   - P99 latency (SLO-driven)

2. Behavior tuning:
   - Scale up: immediate, aggressive (100% or +10 pods)
   - Scale down: slow, conservative (10% per minute, 5min stabilization)

3. KEDA for event-driven workloads:
   - Queue depth triggers
   - Cron-based pre-scaling
   - Scale-to-zero for cost savings

4. VPA for right-sizing:
   - "Off" mode for recommendations only
   - Auto mode for non-critical workloads

5. Pod optimization:
   - Startup probes (not just readiness)
   - Pre-pulled images (DaemonSet trick)
   - Resource requests = actual usage (not guesses)
```
