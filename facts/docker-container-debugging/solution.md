## Step 1: Identify the Exit Code

```bash
# Check the exit code of the crashed container
docker inspect <container> --format='{{.State.ExitCode}}'
docker inspect <container> --format='{{.State.OOMKilled}}'
```

| Exit Code | Meaning | Common Cause |
|-----------|---------|-------------|
| `0` | Normal exit | App finished or `CMD` ended |
| `1` | Application error | Unhandled exception, missing config |
| `126` | Permission denied | Binary not executable |
| `127` | Command not found | Wrong `CMD`/`ENTRYPOINT`, missing binary |
| `137` | SIGKILL (128+9) | **OOMKilled** or `docker kill` |
| `139` | SIGSEGV (128+11) | Segfault, native memory corruption |
| `143` | SIGTERM (128+15) | Graceful shutdown signal |

## Step 2: Read the Logs (Properly)

```bash
# Last 100 lines with timestamps
docker logs --tail 100 --timestamps <container>

# Follow logs in real-time (even on a crashing container)
docker logs -f <container>

# If using docker-compose
docker compose logs --tail 100 <service>
```

**If logs show "Killed" with no other output**: the process was OOMKilled before it could write to stdout. Check Step 3.

## Step 3: Diagnose OOMKilled

```bash
# Check memory limit vs actual usage
docker stats --no-stream <container>

# Check the memory limit set on the container
docker inspect <container> --format='{{.HostConfig.Memory}}'

# Check host system OOM events
dmesg | grep -i "oom\|killed" | tail -20
```

**Common fixes**:
```yaml
# docker-compose.yml: increase memory limit
services:
  app:
    deploy:
      resources:
        limits:
          memory: 512M  # was 256M, app needs more
    environment:
      - NODE_OPTIONS=--max-old-space-size=384  # Node.js heap limit
```

## Step 4: Shell Into a Crashing Container

```bash
# Override the entrypoint to get a shell (even if the app crashes)
docker run -it --entrypoint /bin/sh <image>

# For a stopped container, copy files out for inspection
docker cp <stopped-container>:/app/logs ./debug-logs

# Run the app manually inside the container to see the error
docker run -it --entrypoint /bin/sh <image>
# Then inside:
node server.js  # see the actual error output
```

## Step 5: Common Causes Checklist

- **Missing environment variables**: app crashes on startup because `DATABASE_URL` is undefined
- **Port conflict**: another container already binds to the same port
- **File permissions**: volume mounts owned by root, app runs as non-root user
- **DNS resolution**: container can't reach other services (check network)
- **Health check too aggressive**: container is healthy but health check kills it before startup completes

```bash
# Quick diagnostic commands
docker inspect <container> --format='{{json .State}}' | jq
docker inspect <container> --format='{{json .HostConfig.RestartPolicy}}'
docker network inspect <network>
docker exec <container> env  # check environment variables
docker exec <container> cat /etc/resolv.conf  # check DNS
```
