## Redis Memory Audit Checklist

### 1. Current State Analysis

```bash
# Check current memory usage
redis-cli INFO memory | grep -E "used_memory|evicted"

# Identify large keys
redis-cli --bigkeys

# Check key distribution
redis-cli KEYS '*' | wc -l  # Total key count

# Check TTL distribution
redis-cli SCRIPT LOAD "
local keys = redis.call('KEYS', '*')
local ttls = {}
for _, key in ipairs(keys) do
    local ttl = redis.call('TTL', key)
    table.insert(ttls, ttl)
end
return ttls
" | head -20
```

- [ ] Document current memory: ___GB
- [ ] Document max memory config: ___GB
- [ ] Document eviction policy: ________
- [ ] Identify top 10 largest keys
- [ ] Identify keys with no TTL (TTL = -1)
- [ ] Measure cache hit rate: ___%

### 2. TTL Audit

```bash
# Find keys with NO expiry (dangerous)
redis-cli EVAL "
local keys = redis.call('KEYS', '*')
local no_ttl = {}
for _, key in ipairs(keys) do
    if redis.call('TTL', key) == -1 then
        table.insert(no_ttl, key)
    end
end
return no_ttl
" 0
```

- [ ] Find all keys with NO TTL
- [ ] Categorize by purpose (session, cache, queue, etc.)
- [ ] Set appropriate TTL for each category
- [ ] Test TTL settings in staging

### 3. Data Structure Optimization

```python
# Audit data structure usage
for key_type in ['string', 'hash', 'list', 'set', 'zset']:
    keys = redis_client.execute_command(
        'COMMAND', 'INFO', f'*{key_type}*'
    )
    count = len(keys)
    print(f"{key_type}: {count} keys")
```

- [ ] String keys: Count = ____, Avg size = ____
- [ ] Hash keys: Count = ____, Better than strings? Y/N
- [ ] List keys: Count = ____, Used correctly? Y/N
- [ ] Are JSON strings compressible? Y/N
- [ ] Can strings → hashes? Y/N
- [ ] Can strings → sets (tags)? Y/N

### 4. Compression Evaluation

```python
# Measure compression ratio
import zlib
import json

sample_key = redis_client.randomkey()
value = redis_client.get(sample_key)
original_size = len(value)
compressed_size = len(zlib.compress(value))
ratio = original_size / compressed_size

print(f"Original: {original_size}B")
print(f"Compressed: {compressed_size}B")
print(f"Ratio: {ratio:.1f}x")
```

- [ ] Compression ratio for typical keys: ___x
- [ ] CPU impact acceptable? Y/N
- [ ] Apply compression to: _________

### 5. Eviction Policy Tuning

```bash
# Current policy
redis-cli CONFIG GET maxmemory-policy
# Output: allkeys-lru, volatile-ttl, etc.

# Check eviction stats
redis-cli INFO stats | grep evicted

# Monitor eviction rate (high = thrashing)
watch -n 1 'redis-cli INFO stats | grep evicted'
```

- [ ] Current policy: ___________
- [ ] Is it appropriate for workload? Y/N
- [ ] Eviction rate acceptable? Y/N
- [ ] Consider: allkeys-lru, allkeys-lfu, volatile-ttl
- [ ] Set max memory: ___GB

### 6. High-Memory Keys Investigation

```bash
# For each big key found:
redis-cli MEMORY USAGE key_name
redis-cli TYPE key_name
redis-cli STRLEN key_name (if string)
redis-cli HLEN key_name (if hash)
redis-cli LLEN key_name (if list)
```

For each large key:
- [ ] Key: __________ Size: ____KB
- [ ] Purpose: __________
- [ ] Can be split? Y/N
- [ ] Can be compressed? Y/N
- [ ] Should have shorter TTL? Y/N
- [ ] Can be deleted? Y/N

### 7. Memory Configuration

```bash
# Edit redis.conf or set at runtime
redis-cli CONFIG SET maxmemory 10gb
redis-cli CONFIG SET maxmemory-policy allkeys-lru
redis-cli CONFIG SET maxmemory-samples 5  # For sampling efficiency

# Verify
redis-cli CONFIG GET maxmemory*
```

- [ ] maxmemory set: ___GB
- [ ] maxmemory-policy: __________
- [ ] maxmemory-samples: ____ (5-10 recommended)
- [ ] Tested in staging? Y/N

### 8. Code Changes

```python
# Audit codebase for Redis usage
# Search for: redis_client.set/setex/hset etc.

# Checklist:
# - All cache keys have TTL? Y/N
# - JSON values compressed (if > 500B)? Y/N
# - Using HASH for multi-field data? Y/N
# - Using SET for tags? Y/N
# - Avoiding null/empty values? Y/N
```

- [ ] Audit codebase for memory issues
- [ ] All SET calls include TTL (EX/PX)? Y/N
- [ ] Large JSON values compressed? Y/N
- [ ] Using optimal data structures? Y/N
- [ ] Implement cleanup for orphaned keys? Y/N

### 9. Monitoring Setup

```python
# Add monitoring dashboard
redis_client.info('memory')  # Track in Prometheus/Datadog

# Alert thresholds
- used_memory > maxmemory * 0.8 → Alert
- eviction_rate > 1000/sec → Alert
- cache_hit_rate < 50% → Alert
```

- [ ] Export memory metrics: Y/N
- [ ] Set alerts for high memory: Y/N
- [ ] Monitor eviction rate: Y/N
- [ ] Track cache hit rate: Y/N
- [ ] Dashboard setup: Y/N

### 10. Performance Testing

```bash
# Benchmark before/after
# Before optimization
redis-benchmark -q -l

# After optimization
redis-benchmark -q -l

# Compare:
# - Throughput (ops/sec)
# - Latency (ms)
# - Memory usage
```

- [ ] Benchmark baseline: ____ ops/sec
- [ ] Benchmark optimized: ____ ops/sec
- [ ] Memory reduction: ___% (target: 50%+)
- [ ] Performance impact: ___%
- [ ] Acceptable? Y/N

## Optimization Priorities

1. **Quick wins (< 1 hour)**
   - [ ] Add TTLs to keys without expiry
   - [ ] Set maxmemory + eviction policy
   - [ ] Compress largest keys

2. **Medium effort (< 1 day)**
   - [ ] Convert strings → hashes
   - [ ] Implement compression layer
   - [ ] Add monitoring

3. **Long term (1+ week)**
   - [ ] Refactor code for optimal data structures
   - [ ] Archive cold data
   - [ ] Upgrade Redis version (better compression)

## Post-Optimization Validation

- [ ] Memory usage reduced to target: ___GB
- [ ] Cache hit rate maintained: ___%
- [ ] No performance regression: Y/N
- [ ] No increase in CPU (compression): Y/N
- [ ] Monitoring active and alerting: Y/N
- [ ] Team trained on best practices: Y/N

## Monthly Maintenance

```bash
# Run monthly to detect bloat early
redis-cli INFO memory
redis-cli --bigkeys
redis-cli SCRIPT LOAD "
local keys = redis.call('KEYS', '*')
local no_ttl = {}
for _, key in ipairs(keys) do
    if redis.call('TTL', key) == -1 then
        table.insert(no_ttl, key)
    end
end
return {keys=table.getn(keys), no_ttl=table.getn(no_ttl)}
"

# Alert if:
# - Memory grows > 10% month-over-month
# - Keys with no TTL increase
# - Eviction rate spikes
```
