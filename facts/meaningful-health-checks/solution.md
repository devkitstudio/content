# Deep Health Checks with Dependency Verification

## Three Types of Health Checks

### 1. Liveness (Is the process alive?)
Simple check that the service process is running. Should NOT include dependencies.

```javascript
// LIVENESS - lightweight, fast
app.get('/health/live', (req, res) => {
  // If we can respond, the process is alive
  res.json({ status: 'alive' });
});
```

### 2. Readiness (Can the service handle requests?)
Check that critical dependencies are available.

```javascript
// READINESS - includes dependency checks
app.get('/health/ready', async (req, res) => {
  const checks = {
    database: false,
    cache: false,
    externalApi: false
  };

  // Check database
  try {
    await db.query('SELECT 1');
    checks.database = true;
  } catch (err) {
    logger.error('Database check failed:', err);
  }

  // Check cache (Redis)
  try {
    await redis.ping();
    checks.cache = true;
  } catch (err) {
    logger.error('Cache check failed:', err);
  }

  // Check critical external API
  try {
    await timeout(fetchExternalService(), 5000);
    checks.externalApi = true;
  } catch (err) {
    logger.error('External API check failed:', err);
  }

  const ready = checks.database && checks.cache;
  const status = ready ? 200 : 503;

  res.status(status).json({
    status: ready ? 'ready' : 'not-ready',
    checks
  });
});
```

### 3. Deep Health (Detailed diagnostics)
Comprehensive check for debugging, not for load balancing decisions.

```javascript
// DEEP HEALTH - full diagnostics
app.get('/health/deep', async (req, res) => {
  const timestamp = Date.now();
  const diagnostics = {
    timestamp,
    uptime: process.uptime(),
    memory: process.memoryUsage(),
    checks: {}
  };

  // Database health
  try {
    const start = Date.now();
    const result = await db.query('SELECT 1');
    diagnostics.checks.database = {
      status: 'ok',
      latency: Date.now() - start,
      connection: db.connection?.state
    };
  } catch (err) {
    diagnostics.checks.database = {
      status: 'error',
      error: err.message,
      connection: db.connection?.state
    };
  }

  // Cache (Redis) health
  try {
    const start = Date.now();
    await redis.ping();
    diagnostics.checks.cache = {
      status: 'ok',
      latency: Date.now() - start,
      memory: await redis.info('memory')
    };
  } catch (err) {
    diagnostics.checks.cache = {
      status: 'error',
      error: err.message
    };
  }

  // Message queue health
  try {
    const start = Date.now();
    const stats = await queue.getStats();
    diagnostics.checks.queue = {
      status: 'ok',
      latency: Date.now() - start,
      pending: stats.waiting,
      active: stats.active,
      failed: stats.failed
    };
  } catch (err) {
    diagnostics.checks.queue = {
      status: 'error',
      error: err.message
    };
  }

  // External service health
  try {
    const start = Date.now();
    const response = await timeout(
      fetch('https://api.example.com/health'),
      3000
    );
    diagnostics.checks.externalApi = {
      status: response.ok ? 'ok' : 'degraded',
      latency: Date.now() - start,
      statusCode: response.status
    };
  } catch (err) {
    diagnostics.checks.externalApi = {
      status: 'error',
      error: err.message
    };
  }

  // File system health
  try {
    const start = Date.now();
    const stat = await fs.promises.stat('/tmp');
    diagnostics.checks.filesystem = {
      status: 'ok',
      latency: Date.now() - start
    };
  } catch (err) {
    diagnostics.checks.filesystem = {
      status: 'error',
      error: err.message
    };
  }

  res.json(diagnostics);
});

// Helper: timeout wrapper
function timeout(promise, ms) {
  return Promise.race([
    promise,
    new Promise((_, reject) =>
      setTimeout(() => reject(new Error('Timeout')), ms)
    )
  ]);
}
```

## Common Pitfalls to Avoid

### WRONG: Check only that the process is running
```javascript
// WRONG - says OK even if DB is down
app.get('/health', (req, res) => {
  res.status(200).json({ status: 'ok' });
});
```

### WRONG: Include all checks in liveness probe
```javascript
// WRONG - slow, blocks traffic when dependencies are slow
app.get('/health/live', async (req, res) => {
  // Testing DB, cache, API, filesystem...
  // This takes 10+ seconds, defeating the purpose
  res.json({ status: 'alive' });
});
```

### WRONG: Not timeout on external checks
```javascript
// WRONG - external service hangs, health check hangs forever
app.get('/health/ready', async (req, res) => {
  const response = await fetch('https://slow-external-api.com/status');
  res.json({ ready: response.ok });
});
```

## Kubernetes Probe Configuration

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-service
spec:
  template:
    spec:
      containers:
      - name: api
        image: api:latest

        # Liveness: Is the container alive?
        livenessProbe:
          httpGet:
            path: /health/live
            port: 3000
          initialDelaySeconds: 10
          periodSeconds: 10
          timeoutSeconds: 2
          failureThreshold: 3

        # Readiness: Can it handle traffic?
        readinessProbe:
          httpGet:
            path: /health/ready
            port: 3000
          initialDelaySeconds: 5
          periodSeconds: 5
          timeoutSeconds: 5
          failureThreshold: 2

        # Startup: Did it finish initializing?
        startupProbe:
          httpGet:
            path: /health/live
            port: 3000
          initialDelaySeconds: 0
          periodSeconds: 10
          timeoutSeconds: 2
          failureThreshold: 30 # 30 * 10s = 5 minutes max
```

## Health Check Best Practices

1. **Liveness should be fast** (< 100ms)
2. **Readiness checks critical dependencies only**
3. **Always use timeouts** on external calls
4. **Deep checks for debugging, not for orchestration**
5. **Log health check failures** for debugging
6. **Don't include optional services in readiness**
7. **Cache results** for expensive checks (5-10 seconds)

```javascript
// Cache results to avoid expensive repeated checks
class HealthChecker {
  constructor(cacheDuration = 5000) {
    this.cache = {};
    this.cacheDuration = cacheDuration;
  }

  async check(name, fn) {
    const cached = this.cache[name];
    if (cached && Date.now() - cached.timestamp < this.cacheDuration) {
      return cached.result;
    }

    try {
      const result = { status: 'ok', result: await fn() };
      this.cache[name] = { result, timestamp: Date.now() };
      return result;
    } catch (err) {
      const result = { status: 'error', error: err.message };
      this.cache[name] = { result, timestamp: Date.now() };
      return result;
    }
  }
}
```
