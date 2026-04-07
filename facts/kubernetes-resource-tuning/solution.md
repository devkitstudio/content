# Kubernetes Resource Requests and Limits

## 1. Understanding Requests vs Limits

**Requests:** Guaranteed minimum resources for pod scheduling
**Limits:** Hard ceiling - pod gets killed if exceeded (OOMKill)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-pod
spec:
  containers:
  - name: app
    image: myapp:latest
    resources:
      requests:
        memory: "256Mi"   # Reserve 256Mi (guaranteed)
        cpu: "250m"       # Reserve 0.25 CPU cores
      limits:
        memory: "512Mi"   # Max 512Mi (OOMKill above this)
        cpu: "1000m"      # Max 1 CPU core
```

## 2. Determining Correct Values

**Measure actual usage:**

```bash
# Check current memory usage
kubectl top pod <pod-name> -n <namespace>
kubectl top node

# Check memory over time
kubectl logs -f <pod-name> | grep "memory\|heap"

# Monitor in Prometheus
avg(container_memory_usage_bytes{pod="app"}) / 1024 / 1024
max(container_memory_usage_bytes{pod="app"}) / 1024 / 1024
```

**Typical pattern:** Peak usage = 600Mi, 95th percentile = 500Mi

## 3. Setting Safe Values

**Conservative approach:**

```yaml
resources:
  requests:
    memory: "256Mi"      # 50% of expected average
    cpu: "100m"
  limits:
    memory: "1Gi"        # 1.5-2x peak observed usage
    cpu: "1000m"
```

**Rationale:**
- Request (256Mi) lets scheduler place pods efficiently
- Limit (1Gi) prevents one pod from killing others
- Buffer (400Mi above peak) handles temporary spikes

## 4. Quality of Service (QoS) Classes

**Guaranteed (highest priority, can't evict):**
```yaml
resources:
  requests:
    memory: "512Mi"
    cpu: "500m"
  limits:
    memory: "512Mi"  # Requests == Limits
    cpu: "500m"
```

**Burstable (medium priority):**
```yaml
resources:
  requests:
    memory: "256Mi"
    cpu: "250m"
  limits:
    memory: "1Gi"    # Requests != Limits (can burst)
    cpu: "1000m"
```

**BestEffort (no requests/limits - evicted first):**
```yaml
# No resource section = lowest priority
# Gets evicted when node pressure occurs
```

## 5. Handling Memory Spikes

**Identify spike causes:**

```javascript
// Node.js example: memory leak detection
const heapdump = require('heapdump');
const os = require('os');

setInterval(() => {
  const used = process.memoryUsage();
  const totalMem = os.totalmem();

  if (used.heapUsed / totalMem > 0.8) {
    console.error('High memory usage:', {
      heapUsed: Math.round(used.heapUsed / 1024 / 1024) + 'MB',
      heapTotal: Math.round(used.heapTotal / 1024 / 1024) + 'MB',
      external: Math.round(used.external / 1024 / 1024) + 'MB',
    });

    // Dump heap for analysis
    heapdump.writeSnapshot();
  }
}, 10000);
```

**Common causes:**
- Memory leaks (retained objects)
- Unbounded caches
- Queue accumulation
- Connection pool leaks

## 6. Safe Limit Configuration

**Avoid OOMKill with proper buffer:**

```yaml
# BAD: Limit too close to peak
resources:
  requests:
    memory: "256Mi"
  limits:
    memory: "600Mi"  # Peak is 600Mi, no buffer!
    # Result: Random OOMKills on spikes

# GOOD: Reasonable buffer
resources:
  requests:
    memory: "256Mi"
  limits:
    memory: "1Gi"    # 400Mi buffer above peak
    # Result: No OOMKills from normal operation
```

## 7. CPU Settings (Less Critical)

**CPU is compressible (pods just get less CPU):**

```yaml
resources:
  requests:
    cpu: "100m"      # Scheduler: need at least 0.1 cores
  limits:
    cpu: "500m"      # Throttle above 0.5 cores (not kill)
```

**CPU limits cause slowdown, not crashes:**
- Unlike memory, CPU doesn't hard-kill
- Pods get less CPU time when limit exceeded
- Use for fairness, not correctness

## 8. Multi-Container Pod Resources

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: multi-container
spec:
  containers:
  # Container 1
  - name: app
    image: myapp:latest
    resources:
      requests:
        memory: "256Mi"
        cpu: "250m"
      limits:
        memory: "1Gi"
        cpu: "1000m"

  # Container 2
  - name: sidecar
    image: sidecar:latest
    resources:
      requests:
        memory: "64Mi"
        cpu: "50m"
      limits:
        memory: "256Mi"
        cpu: "200m"

  # Pod total: 320Mi requested, 1.25Gi limit
```

## Resource Sizing Checklist

- [ ] Measured actual memory usage (min/avg/p95/peak)
- [ ] Requests set to 80% of observed average
- [ ] Limits set to 1.5-2x observed peak
- [ ] 20-30% buffer above peak for safety
- [ ] CPU requests based on baseline workload
- [ ] QoS class appropriate for workload (Guaranteed for critical)
- [ ] Vertical Pod Autoscaler (VPA) configured for tuning
- [ ] HPA (Horizontal Pod Autoscaler) for scaling

