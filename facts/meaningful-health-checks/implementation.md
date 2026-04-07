# Complete Implementation: Liveness vs Readiness vs Startup Probes

## Express.js Service with All Three Probes

```javascript
const express = require('express');
const db = require('./database');
const redis = require('./redis');
const { queue } = require('./queue');

const app = express();

// =====================================================================
// LIVENESS PROBE - Tells orchestrator if process is alive
// Used by: Kubernetes, Docker Swarm, health checkers
// Response: 200 OK if process responds, timeout/5xx if not
// Speed: Must be < 1 second
// =====================================================================

app.get('/health/live', (req, res) => {
  // Respond immediately if process is running
  res.status(200).json({
    status: 'alive',
    timestamp: new Date().toISOString(),
    uptime: process.uptime()
  });
});

// =====================================================================
// READINESS PROBE - Tells orchestrator if ready for traffic
// Used by: Kubernetes load balancer, traffic routers
// Response: 200 OK if ready, 503 if not ready
// Speed: Can be 5-10 seconds, but shouldn't block
// =====================================================================

const readinessChecks = {
  database: null,
  cache: null,
  timestamp: null
};

// Cache results for 5 seconds
async function checkReadiness() {
  const now = Date.now();
  if (readinessChecks.timestamp && now - readinessChecks.timestamp < 5000) {
    return {
      database: readinessChecks.database,
      cache: readinessChecks.cache
    };
  }

  // Check database connectivity
  let dbReady = false;
  try {
    await db.query('SELECT 1 as health');
    dbReady = true;
  } catch (err) {
    console.error('Database check failed:', err.message);
    dbReady = false;
  }

  // Check cache connectivity
  let cacheReady = false;
  try {
    await redis.ping();
    cacheReady = true;
  } catch (err) {
    console.error('Cache check failed:', err.message);
    cacheReady = false;
  }

  // Cache the results
  readinessChecks.database = dbReady;
  readinessChecks.cache = cacheReady;
  readinessChecks.timestamp = now;

  return { database: dbReady, cache: cacheReady };
}

app.get('/health/ready', async (req, res) => {
  const checks = await checkReadiness();

  // Only check critical dependencies
  const isReady = checks.database && checks.cache;

  res.status(isReady ? 200 : 503).json({
    status: isReady ? 'ready' : 'not-ready',
    checks,
    timestamp: new Date().toISOString()
  });
});

// =====================================================================
// STARTUP PROBE - Tells orchestrator if initialization complete
// Used by: Kubernetes startup checks (useful for slow startup)
// Response: 200 OK when ready, anything else = still starting
// =====================================================================

let initializationComplete = false;

app.get('/health/startup', (req, res) => {
  if (initializationComplete) {
    res.status(200).json({
      status: 'started',
      timestamp: new Date().toISOString()
    });
  } else {
    res.status(503).json({
      status: 'starting',
      timestamp: new Date().toISOString()
    });
  }
});

// Initialization logic
async function initialize() {
  try {
    console.log('Initializing service...');

    // Connect to database
    console.log('Connecting to database...');
    await db.connect();
    console.log('Database connected');

    // Connect to cache
    console.log('Connecting to cache...');
    await redis.connect();
    console.log('Cache connected');

    // Setup message queue
    console.log('Setting up message queue...');
    await queue.setup();
    console.log('Message queue ready');

    // Load configuration
    console.log('Loading configuration...');
    await loadConfiguration();
    console.log('Configuration loaded');

    initializationComplete = true;
    console.log('Service initialization complete');
  } catch (err) {
    console.error('Initialization failed:', err);
    process.exit(1);
  }
}

// =====================================================================
// DEEP HEALTH CHECK - For manual debugging and monitoring dashboards
// =====================================================================

app.get('/health/deep', async (req, res) => {
  const report = {
    timestamp: new Date().toISOString(),
    uptime: process.uptime(),
    checks: {}
  };

  // Database diagnostics
  const dbStart = Date.now();
  try {
    await db.query('SELECT 1 as health');
    report.checks.database = {
      status: 'healthy',
      latency: Date.now() - dbStart,
      pool: {
        total: db.pool?.totalCount || 0,
        idle: db.pool?.idleCount || 0,
        waiting: db.pool?.waitingCount || 0
      }
    };
  } catch (err) {
    report.checks.database = {
      status: 'unhealthy',
      error: err.message,
      latency: Date.now() - dbStart
    };
  }

  // Cache diagnostics
  const cacheStart = Date.now();
  try {
    await redis.ping();
    const info = await redis.info('memory');
    report.checks.cache = {
      status: 'healthy',
      latency: Date.now() - cacheStart,
      memory: parseRedisInfo(info)
    };
  } catch (err) {
    report.checks.cache = {
      status: 'unhealthy',
      error: err.message,
      latency: Date.now() - cacheStart
    };
  }

  // Queue diagnostics
  try {
    const stats = await queue.getStats();
    report.checks.queue = {
      status: 'healthy',
      jobs: {
        waiting: stats.waiting,
        active: stats.active,
        completed: stats.completed,
        failed: stats.failed,
        delayed: stats.delayed
      }
    };
  } catch (err) {
    report.checks.queue = {
      status: 'unhealthy',
      error: err.message
    };
  }

  // Memory usage
  const memUsage = process.memoryUsage();
  report.memory = {
    rss: `${(memUsage.rss / 1024 / 1024).toFixed(2)} MB`,
    heapTotal: `${(memUsage.heapTotal / 1024 / 1024).toFixed(2)} MB`,
    heapUsed: `${(memUsage.heapUsed / 1024 / 1024).toFixed(2)} MB`,
    external: `${(memUsage.external / 1024 / 1024).toFixed(2)} MB`
  };

  res.json(report);
});

// =====================================================================
// Startup
// =====================================================================

const PORT = process.env.PORT || 3000;

async function start() {
  await initialize();

  app.listen(PORT, () => {
    console.log(`Server running on port ${PORT}`);
  });
}

start().catch(err => {
  console.error('Failed to start server:', err);
  process.exit(1);
});

// =====================================================================
// Graceful shutdown
// =====================================================================

process.on('SIGTERM', async () => {
  console.log('Received SIGTERM, shutting down gracefully...');

  // Mark as not ready - stop accepting new requests
  initializationComplete = false;

  // Wait a bit for load balancers to notice
  await new Promise(resolve => setTimeout(resolve, 5000));

  // Close database connections
  await db.close();

  // Close cache connections
  await redis.close();

  process.exit(0);
});
```

## Helper Functions

```javascript
function parseRedisInfo(info) {
  const lines = info.split('\r\n');
  const result = {};

  lines.forEach(line => {
    if (line && !line.startsWith('#')) {
      const [key, value] = line.split(':');
      result[key] = value;
    }
  });

  return {
    usedMemory: `${(parseInt(result.used_memory) / 1024 / 1024).toFixed(2)} MB`,
    peakMemory: `${(parseInt(result.used_memory_peak) / 1024 / 1024).toFixed(2)} MB`,
    memoryFragmentation: result.mem_fragmentation_ratio
  };
}

async function loadConfiguration() {
  // Simulate loading configuration
  return new Promise(resolve => setTimeout(resolve, 100));
}
```

## Monitoring & Alerting

```javascript
// Prometheus metrics for health checks
const prometheus = require('prom-client');

const healthCheckLatency = new prometheus.Histogram({
  name: 'health_check_latency_ms',
  help: 'Health check response time',
  labelNames: ['check_type']
});

const healthCheckStatus = new prometheus.Gauge({
  name: 'health_check_status',
  help: '1 = healthy, 0 = unhealthy',
  labelNames: ['check_name']
});

// Track metrics
app.get('/health/ready', async (req, res) => {
  const start = Date.now();
  const checks = await checkReadiness();
  const latency = Date.now() - start;

  healthCheckLatency.observe({ check_type: 'readiness' }, latency);
  healthCheckStatus.set({ check_name: 'database' }, checks.database ? 1 : 0);
  healthCheckStatus.set({ check_name: 'cache' }, checks.cache ? 1 : 0);

  res.json({ checks });
});

app.get('/metrics', async (req, res) => {
  res.set('Content-Type', prometheus.register.contentType);
  res.end(await prometheus.register.metrics());
});
```
