## How OOM Killer Works

When system runs out of memory:

```
1. Kernel detects: free_memory < min_threshold
2. Triggers OOM killer
3. Scans all processes, calculates OOM score
4. Kills process with highest score
5. Frees memory from that process
```

**OOM Score Calculation:**

```
oom_score = (resident_memory / total_memory) * 1000 + adjustment

Example on 16GB system:
- Process A: 2GB RAM → score = (2/16)*1000 = 125
- Process B: 8GB RAM → score = (8/16)*1000 = 500

OOM killer kills Process B (highest score)
```

## OOM Score Adjustment (oom_score_adj)

```bash
# View OOM score for a process
cat /proc/$(pidof postgres)/oom_score
# Output: 523

# View adjustment (default 0)
cat /proc/$(pidof postgres)/oom_score_adj
# Output: 0
```

**Adjustment Range: -1000 to +1000**

```
final_oom_score = base_oom_score + adjustment

# Protect critical processes
sudo sh -c 'echo -900 > /proc/$(pidof postgres)/oom_score_adj'
# Now: 523 + (-900) = capped at 0 (won't be killed)

# Mark non-critical processes as safe to kill
sudo sh -c 'echo 500 > /proc/$(pidof memcached)/oom_score_adj'
# Now: 200 + 500 = 700 (higher priority for killing)
```

## Permanent Configuration

```bash
# Create systemd service file for PostgreSQL
cat > /etc/systemd/system/postgresql.service.d/oom-protection.conf <<EOF
[Service]
OOMScoreAdjust=-900
EOF

sudo systemctl daemon-reload
sudo systemctl restart postgresql

# Verify
cat /proc/$(pidof postgres)/oom_score_adj
# Output: -900
```

## Using cgroups (Memory Limits)

```bash
# Create cgroup for application
sudo cgcreate -g memory:/app

# Limit memory to 4GB (prevents runaway)
echo 4294967296 | sudo tee /sys/fs/cgroup/memory/app/memory.limit_in_bytes

# Add process to cgroup
echo $PID | sudo tee /sys/fs/cgroup/memory/app/cgroup.procs

# Check memory usage
cat /sys/fs/cgroup/memory/app/memory.usage_in_bytes
```

## systemd MemoryLimit

```ini
# /etc/systemd/system/myapp.service
[Unit]
Description=My Application
After=network.target

[Service]
Type=simple
ExecStart=/usr/bin/myapp
Restart=on-failure

# Prevent OOM kills
OOMScoreAdjust=-900

# Also set memory limit (cgroup)
MemoryLimit=4G
MemoryAccounting=yes

[Install]
WantedBy=multi-user.target
```

## Multi-Tier Protection

```bash
# Tier 1: Critical system services (won't be killed)
# PostgreSQL, Redis, Nginx, systemd-journald
for proc in postgres redis-server nginx; do
  pid=$(pidof $proc)
  if [ ! -z "$pid" ]; then
    echo -900 | sudo tee /proc/$pid/oom_score_adj > /dev/null
  fi
done

# Tier 2: Important app services (protected but killable)
# Application servers, load balancers
echo -500 | sudo tee /proc/$(pidof java)/oom_score_adj > /dev/null

# Tier 3: Temporary processes (kill first)
# Batch jobs, cache builders
echo 500 | sudo tee /proc/$(pidof batch_job)/oom_score_adj > /dev/null
```

## Monitoring OOM Events

```bash
# Check if OOM occurred (in dmesg)
sudo dmesg | grep -i "out of memory" | tail -5

# Real-time monitoring
sudo journalctl -f -u kernel | grep -i "oom"

# Look for OOM killer invocations
sudo journalctl --all | grep "Out of memory"
```

## Prevention Strategies

### 1. Memory Monitoring

```bash
# Check system memory usage
free -h

# Check per-process usage
ps aux --sort=-%mem | head -20

# Watch memory in real-time
watch -n 1 'free -h && echo && ps aux --sort=-%mem | head -10'
```

### 2. Set Swap Space

```bash
# Create 8GB swap file
sudo fallocate -l 8G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile

# Verify
swapon --show

# Persist (add to /etc/fstab)
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
```

### 3. Memory Pressure Notifications

```bash
# Enable memory pressure notifications
echo 70 | sudo tee /proc/sys/vm/memory_pressure_warning > /dev/null

# This alerts before OOM killer activates
```

## Complete Protection Script

```bash
#!/bin/bash
# protect-critical-processes.sh

CRITICAL_PROCESSES=("postgres" "redis-server" "nginx" "mongod")
IMPORTANT_PROCESSES=("java" "python3" "node")

# Protect critical processes
for proc in "${CRITICAL_PROCESSES[@]}"; do
  pid=$(pidof "$proc")
  if [ ! -z "$pid" ]; then
    echo "Protecting $proc (PID: $pid)"
    echo -900 | sudo tee /proc/$pid/oom_score_adj > /dev/null
  fi
done

# Set moderate protection for important processes
for proc in "${IMPORTANT_PROCESSES[@]}"; do
  pid=$(pgrep -f "$proc" | head -1)
  if [ ! -z "$pid" ]; then
    echo "Protecting $proc (PID: $pid)"
    echo -500 | sudo tee /proc/$pid/oom_score_adj > /dev/null
  fi
done

echo "OOM protection configured"
```

## Checking Current Scores

```bash
# All processes and their OOM scores
ps aux --sort=-%mem | while read user pid rest; do
  if [ "$pid" = "PID" ]; then
    echo "$user PID OOM_SCORE OOM_ADJ COMMAND"
  else
    oom=$(cat /proc/$pid/oom_score 2>/dev/null)
    adj=$(cat /proc/$pid/oom_score_adj 2>/dev/null)
    echo "$user $pid $oom $adj $(ps -p $pid -o comm=)"
  fi
done | head -20
```
