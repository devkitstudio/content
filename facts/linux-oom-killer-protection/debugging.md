## Diagnosing OOM Kills

### From dmesg

```bash
# View kernel message buffer
sudo dmesg | tail -100

# Look for OOM killer output
sudo dmesg | grep -A 30 "Out of memory"

# Example output:
# [12345.678901] Out of memory: Kill process postgres (1234) score 523 or sacrifice child
# [12345.678902] Killed process postgres (1234) total-vm:2097152kB, anon-rss:1048576kB, file-rss:0kB, shmem-rss:0kB
```

### From journalctl

```bash
# Recent OOM events
sudo journalctl --since "1 hour ago" | grep -i "oom"

# Follow real-time
sudo journalctl -f | grep --line-buffered -i "oom"

# Pretty print with timestamps
sudo journalctl --since "1 day ago" | grep "Out of memory" | \
  while read line; do
    echo "$(echo $line | cut -d' ' -f1-2): OOM Event"
  done
```

## Analyzing Memory Usage

### Top Memory Consumers

```bash
# Sort by RSS (resident memory)
ps aux --sort=-%mem | head -20

# Get detailed memory breakdown
ps -eo pid,user,%mem,rss,vsz,comm --sort=-%mem | head -20

# Output:
#  PID  USER  %MEM    RSS      VSZ COMM
# 1234 postgres 50.0 8388608 16777216 postgres
# 5678 nginx    10.0 1677721 3355443 nginx
```

### Per-Process Memory Details

```bash
# Check process memory map
cat /proc/1234/status | grep -E "^Vm|^Rss"
# Output:
# VmPeak:  16777216 kB
# VmSize:  16777216 kB
# VmRSS:    8388608 kB

# Check memory pressure
cat /proc/pressure/memory
# Output:
# some avg10=5.00 avg60=3.20 avg300=2.10 total=123456789
# full avg10=0.50 avg60=0.30 avg300=0.20 total=23456789
```

### Free Memory Trend

```bash
# Create a memory trend log
watch -n 5 'date >> /tmp/memory.log; free >> /tmp/memory.log'

# Or automated script
while true; do
  echo "$(date '+%Y-%m-%d %H:%M:%S') $(free -h | grep Mem | awk '{print $3 " / " $2}')" \
    >> /tmp/memory_usage.log
  sleep 60
done
```

## Root Cause Analysis

### Is it a memory leak?

```bash
# Monitor a process over time
for i in {1..10}; do
  mem=$(ps -o rss= -p $PID)
  timestamp=$(date '+%Y-%m-%d %H:%M:%S')
  echo "$timestamp PID $PID Memory: ${mem}KB"
  sleep 60
done

# If constantly growing → memory leak
# If stable → normal operation
# If sudden spike → cache flush or query spike
```

### Is it caching?

```bash
# Check available cache memory
cat /proc/meminfo | grep -E "Cached|Buffers"
# Output:
# Buffers:         1024 kB
# Cached:       524288 kB

# Total usable = Free + Buffers + Cached
free_actual=$(awk '/MemFree/ {print $2}' /proc/meminfo)
cached=$(awk '/Cached/ {print $2}' /proc/meminfo)
buffers=$(awk '/Buffers/ {print $2}' /proc/meminfo)
total_available=$((free_actual + cached + buffers))

echo "Available: $total_available KB"
```

### Check for swap thrashing

```bash
# Monitor swap usage
watch -n 1 'free -h | tail -2'

# If swap usage is high and increasing → thrashing
# Each process waiting for disk I/O → system slow

# Check swap I/O statistics
cat /proc/vmstat | grep -E "pswpin|pswpout"
# pswpin = pages swapped in (from disk)
# pswpout = pages swapped out (to disk)
# High values = excessive swap activity
```

## Historical Analysis

### Enable OOM event logging

```bash
# Create a log file for OOM events
sudo touch /var/log/oom-events.log
sudo chmod 666 /var/log/oom-events.log

# Create a cronjob to check
*/5 * * * * dmesg | grep -i "Out of memory" >> /var/log/oom-events.log

# Or use systemd timer
cat > /etc/systemd/system/oom-monitor.service <<EOF
[Unit]
Description=Monitor OOM Events
After=network.target

[Service]
Type=simple
ExecStart=/usr/local/bin/oom-monitor.sh
Restart=always

[Install]
WantedBy=multi-user.target
EOF
```

### Timeline reconstruction

```bash
# When did last OOM occur?
sudo dmesg -T | grep -i "out of memory" | tail -1

# What was running then?
# Check logs from that time window
sudo journalctl --since "2024-01-15 10:00:00" --until "2024-01-15 11:00:00"

# What was the system state?
# Check monitoring data (Prometheus, Grafana)
# Memory usage graph around that time
```

## Prevention: Alerting Setup

```bash
# Alert when memory usage > 80%
#!/bin/bash
THRESHOLD=80
CURRENT=$(free | grep Mem | awk '{printf("%d", ($3/$2) * 100)}')

if [ $CURRENT -gt $THRESHOLD ]; then
  echo "ALERT: Memory usage at ${CURRENT}%" | mail -s "Memory Alert" admin@example.com
fi

# Add to crontab
*/5 * * * * /usr/local/bin/memory-alert.sh
```

## Quick Commands Reference

```bash
# Current state
free -h                                    # Memory summary
ps aux --sort=-%mem | head -20             # Top memory processes
cat /proc/pressure/memory                  # Memory pressure
cat /proc/vmstat | tail -20                # VM statistics

# Historical data
sudo dmesg | tail -50                      # Kernel messages
sudo journalctl -p err --since "1 day ago" # Error logs
cat /var/log/messages | grep oom           # System logs
```
