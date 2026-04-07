# Diagnosing OOMKill and Right-Sizing

## 1. Detecting OOMKill

**Check pod status:**

```bash
kubectl describe pod <pod-name> -n <namespace>

# Look for:
# State:          Terminated
#   Reason:       OOMKilled
#   Exit Code:    137
#   Signal:       9
```

**Check events:**

```bash
kubectl get events -n <namespace> --sort-by='.lastTimestamp' | grep OOMKill
```

**Check container logs:**

```bash
# Last logs before crash
kubectl logs <pod-name> --previous

# Real-time monitoring
kubectl logs -f <pod-name>
```

## 2. Memory Usage Investigation

**Get memory metrics:**

```bash
# Single pod current usage
kubectl top pod <pod-name>

# Pod over time (requires metrics-server)
kubectl get --all-namespaces \
  horizontalpodautoscaler.autoscaling | grep -v TARGETS

# Check container memory in detail
kubectl exec <pod-name> -- ps aux
kubectl exec <pod-name> -- cat /proc/self/status | grep VmRSS
```

**Export metrics to analyze:**

```bash
# Using kubectl-resource-capacity plugin
kubectl resource-capacity --util

# Manual query via Prometheus
kubectl port-forward -n prometheus svc/prometheus 9090:9090
# Query: avg(rate(container_memory_usage_bytes{pod="app"}[5m])) by (pod)
```

## 3. Memory Profiling

**Node.js (V8 heap snapshot):**

```javascript
const http = require('http');
const heapdump = require('heapdump');

// Trigger heap dump via HTTP
http.createServer((req, res) => {
  if (req.url === '/heap-dump') {
    heapdump.writeSnapshot(`./heap-${Date.now()}.heapsnapshot`);
    res.end('Heap dump written');
  }
  res.end('OK');
}).listen(3000);

// In container: curl localhost:3000/heap-dump
// Download and analyze in Chrome DevTools
```

**Python (memory_profiler):**

```python
from memory_profiler import profile

@profile
def expensive_function():
    large_list = [i for i in range(10000000)]
    return sum(large_list)

# Run with: python -m memory_profiler script.py
```

**Java (JVM heap):**

```bash
# Get JVM heap dump
kubectl exec <pod-name> -- \
  jmap -dump:live,format=b,file=/tmp/heap.bin 1

# Copy out of pod
kubectl cp <pod-name>:/tmp/heap.bin ./heap.bin

# Analyze with Eclipse MAT or JProfiler
```

## 4. Identifying Memory Leaks

**Monitor growth over time:**

```bash
# Prometheus query: memory growth
rate(container_memory_usage_bytes{pod="app"}[1h])

# If consistently growing = memory leak
# If stable = normal operation
```

**Common culprits:**

1. **Unbounded cache:**
```javascript
// BAD: cache grows forever
const cache = {};
app.get('/user/:id', (req, res) => {
  if (!cache[req.params.id]) {
    cache[req.params.id] = fetchUser(req.params.id);
  }
  res.json(cache[req.params.id]);
});

// GOOD: cache with TTL
const NodeCache = require('node-cache');
const cache = new NodeCache({ stdTTL: 3600 });
```

2. **Retained event listeners:**
```javascript
// BAD: listener never removed
socket.on('data', handler);

// GOOD: remove listener
socket.off('data', handler);
// Or use once:
socket.once('data', handler);
```

3. **Database connection pool leaks:**
```javascript
// BAD: connections never returned to pool
const conn = await pool.getConnection();
// Forgot to release!

// GOOD: always release
try {
  const conn = await pool.getConnection();
  const result = await conn.query('SELECT ...');
  conn.release();
  return result;
} catch (e) {
  conn.release();
  throw e;
}
```

## 5. Right-Sizing Workflow

**Step 1: Establish baseline (week of monitoring)**

```bash
# Watch metrics in Prometheus
container_memory_usage_bytes{pod="myapp"}

# Record:
# - Minimum: 80Mi
# - Average: 250Mi
# - 95th percentile: 400Mi
# - Peak: 600Mi
```

**Step 2: Calculate recommendations**

```
requests = avg * 1.2 = 250Mi * 1.2 = 300Mi
limits = peak * 1.5 = 600Mi * 1.5 = 900Mi

Conservative: requests 256Mi, limits 1Gi
```

**Step 3: Apply and verify**

```yaml
resources:
  requests:
    memory: "256Mi"
  limits:
    memory: "1Gi"
```

**Step 4: Monitor for 1-2 weeks**

```bash
kubectl top pod <pod-name> -n <namespace> --containers

# Verify:
# - No OOMKills
# - Peak usage < limits
# - Average usage < requests
```

## 6. Vertical Pod Autoscaler (VPA)

**Automatic right-sizing:**

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: app-vpa
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: app
  updatePolicy:
    updateMode: "Auto"  # Automatically adjust
  resourcePolicy:
    containerPolicies:
    - containerName: app
      minAllowed:
        memory: "128Mi"
      maxAllowed:
        memory: "2Gi"
      controlledResources: ["memory", "cpu"]
```

**VPA workflow:**
1. Observes actual resource usage
2. Recommends new requests/limits
3. Auto-restarts pods with new values
4. Continuously learns and adjusts

## 7. Troubleshooting Checklist

| Issue | Solution |
|-------|----------|
| Frequent OOMKills | Increase limits by 50%, check for memory leaks |
| High CPU throttling | Increase CPU requests/limits |
| Pod not scheduling | Reduce requests (scheduler can't find node) |
| Unused memory | Reduce requests (waste of cluster resources) |
| Sporadic OOMKills | Add buffer above peak usage |
| Memory leak suspected | Profile with heapdump/jmap, check growth rate |

