## Beyond `nginx -s reload`

### Binary Upgrade (Zero-Downtime Nginx Version Update)

When you need to upgrade Nginx itself (not just config), use binary hot-swap:

```bash
# 1. Build/install new Nginx binary (don't stop the old one)
# The new binary must be at the same path

# 2. Send USR2 to the master process → starts a new master with new binary
kill -USR2 $(cat /var/run/nginx.pid)

# Now you have TWO master processes running:
# Old master (PID in /var/run/nginx.pid.oldbin)
# New master (PID in /var/run/nginx.pid)

# 3. Gracefully shut down old workers
kill -WINCH $(cat /var/run/nginx.pid.oldbin)

# 4. Old workers finish existing connections, then exit
# If everything works, quit the old master:
kill -QUIT $(cat /var/run/nginx.pid.oldbin)

# If something is wrong, rollback:
# kill -HUP $(cat /var/run/nginx.pid.oldbin)  # restart old workers
# kill -QUIT $(cat /var/run/nginx.pid)          # kill new master
```

### Kubernetes: Rolling Update Strategy

In Kubernetes, Nginx reload is handled by the rolling update strategy:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 0        # never kill a pod before new one is ready
      maxSurge: 1              # create 1 extra pod during rollout
  template:
    spec:
      containers:
        - name: nginx
          image: nginx:1.27
          lifecycle:
            preStop:
              exec:
                command: ["/bin/sh", "-c", "nginx -s quit && sleep 15"]
              # Wait for in-flight requests to finish
          readinessProbe:
            httpGet:
              path: /healthz
              port: 80
            initialDelaySeconds: 5
            periodSeconds: 5
      terminationGracePeriodSeconds: 30
```

**Key**: `preStop` hook with `nginx -s quit` ensures graceful shutdown, and `maxUnavailable: 0` ensures no capacity loss.

### ConfigMap Reload with inotify

For config-only changes in Kubernetes, avoid full redeployment:

```yaml
# Mount ConfigMap as volume
volumes:
  - name: nginx-config
    configMap:
      name: nginx-conf

# Use a sidecar to watch for config changes and reload
containers:
  - name: nginx
    image: nginx:1.27
    volumeMounts:
      - name: nginx-config
        mountPath: /etc/nginx/conf.d

  - name: reload-sidecar
    image: alpine:3.19
    command: ["/bin/sh", "-c"]
    args:
      - |
        apk add inotify-tools;
        while inotifywait -e modify /etc/nginx/conf.d/; do
          nginx -t && nginx -s reload;
          echo "Config reloaded at $(date)";
        done
    volumeMounts:
      - name: nginx-config
        mountPath: /etc/nginx/conf.d
```

### HAProxy: Hitless Reload Comparison

HAProxy (an Nginx alternative) has a different approach worth mentioning in interviews:

```bash
# HAProxy uses socket-based seamless reload
# The new process inherits listening sockets from the old one
haproxy -f /etc/haproxy/haproxy.cfg -p /var/run/haproxy.pid -sf $(cat /var/run/haproxy.pid)

# -sf = "finish" → old process finishes connections, then exits
# -st = "terminate" → old process drops connections immediately
```

**HAProxy vs Nginx reload**: HAProxy's seamless reload transfers sockets between processes at the OS level (using `SO_REUSEPORT`), meaning zero connection drops. Nginx reload creates new workers alongside old ones, which is slightly less seamless but still zero-downtime for HTTP.

## Interview Decision Tree

```
Q: "How do you restart Nginx without downtime?"

Level 1 (Junior):
  → "Use nginx -s reload instead of restart"

Level 2 (Mid):
  → "Test config first with nginx -t, then reload"
  → "Reload sends SIGHUP to master, spawns new workers,
     old workers finish existing connections"

Level 3 (Senior):
  → "For binary upgrades, use USR2 signal for hot-swap"
  → "In Kubernetes, use rolling updates with preStop hooks"
  → "For config changes, use ConfigMap + inotify sidecar"
  → "Consider alternatives like HAProxy with socket transfer"

Level 4 (Staff):
  → "Design the entire zero-downtime deployment pipeline"
  → "Blue/green deployments at the load balancer level"
  → "Canary releases with gradual traffic shifting"
```
