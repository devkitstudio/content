## Real Production Incidents

### The Silent OOM

**Symptom**: Node.js container restarts every 5 minutes. Logs show nothing.

**Root cause**: A memory leak in a WebSocket handler. Each connected client stored their full message history in memory. With 2000 concurrent users, heap exceeded the 512MB limit.

**Fix**: Stream messages from Redis instead of in-memory arrays. Added `--max-old-space-size` to match container limit so Node.js GC could be more aggressive.

```dockerfile
# Before: no heap limit, Node defaults to ~1.5GB (exceeds container limit)
CMD ["node", "server.js"]

# After: explicitly match container memory
CMD ["node", "--max-old-space-size=384", "server.js"]
```

### The Phantom Port Conflict

**Symptom**: Container starts, runs for exactly 30 seconds, then exits with code 1. Works perfectly on dev machine.

**Root cause**: Health check was hitting `localhost:3000/health`. The app listened on `0.0.0.0:3000` but the health check ran **before** the app finished connecting to the database. The app returned 503 during warmup, and after 3 failed checks (10s interval), Docker killed it.

**Fix**: Add a startup grace period:
```yaml
healthcheck:
  test: ["CMD", "curl", "-f", "http://localhost:3000/health"]
  interval: 10s
  timeout: 5s
  retries: 5
  start_period: 30s  # ← give the app 30s to warm up before checking
```

### The Volume Permission Trap

**Symptom**: Container crashes immediately with `EACCES: permission denied, open '/app/data/cache.json'`.

**Root cause**: Dockerfile used `USER node` (uid 1000) but the volume mount was owned by root. The app couldn't write to the mounted directory.

**Fix**:
```dockerfile
# Ensure the directory exists and is owned by the app user
RUN mkdir -p /app/data && chown node:node /app/data
USER node
```

Or in docker-compose:
```yaml
volumes:
  - ./data:/app/data:rw
user: "1000:1000"  # match the host user
```

### The DNS Resolution Blackhole

**Symptom**: App starts, then fails with `ECONNREFUSED` when connecting to `postgres:5432`. But `postgres` container is healthy.

**Root cause**: The app container was on the `default` bridge network, while postgres was on a custom `backend` network. Containers on different networks can't resolve each other's hostnames.

**Fix**:
```yaml
services:
  app:
    networks: [backend]  # same network as postgres
  postgres:
    networks: [backend]

networks:
  backend:
    driver: bridge
```
