## Redis Memory Usage Breakdown

```
Current: 32GB for 10M keys
- Average per key: 3.2KB (too large)
- Cache hit rate: 60% (20% should hit, rest are wasted)
- Memory waste: ~10GB+ (old session data)

Target: 8-10GB for same data
- Average per key: 1KB (much better)
- Higher hit rate: Focus on active data only
```

## Strategy 1: Eviction Policies (Active)

Automatically remove least-used keys:

```bash
# redis.conf
maxmemory 10gb
maxmemory-policy allkeys-lru  # Remove least recently used

# Restart or update at runtime
redis-cli CONFIG SET maxmemory 10gb
redis-cli CONFIG SET maxmemory-policy allkeys-lru
```

**Eviction Policies:**
- `noeviction`: Refuse writes if full (default)
- `allkeys-lru`: Remove least recently used key (ANY key)
- `allkeys-lfu`: Remove least frequently used key (better for working set)
- `volatile-lru`: Remove LRU key WITH expiry only
- `volatile-ttl`: Remove key closest to expiry
- `volatile-lfu`: Remove LFU key WITH expiry

**Recommendation:**
```bash
# For cache: Use LRU (most common)
maxmemory-policy allkeys-lru

# For session store: Use LFU (newer Redis 4+)
maxmemory-policy allkeys-lfu

# For temp data: Use TTL-based
maxmemory-policy volatile-ttl
```

## Strategy 2: Set Aggressive TTLs

Most Redis data should expire:

```python
import redis

redis_client = redis.Redis()

# Cache: 1 hour
redis_client.setex('user:123:cache', 3600, data)

# Session: 24 hours
redis_client.setex('session:abc', 86400, session_data)

# Rate limiter: 1 minute
redis_client.setex('ratelimit:user:123', 60, count)

# Temp: 5 minutes
redis_client.setex('temp:job:xyz', 300, result)

# Check average TTL
redis_client.ttl('key_name')  # Returns seconds remaining
```

**Audit existing keys:**
```bash
# Find keys with no expiry (dangerous)
redis-cli KEYS '*' | while read key; do
    ttl=$(redis-cli TTL "$key")
    if [ "$ttl" -eq -1 ]; then
        echo "No TTL: $key"
    fi
done

# Set default TTL for keys without one
for key in $(redis-cli KEYS '*' | grep -v expiring); do
    redis-cli EXPIRE "$key" 3600  # 1 hour
done
```

## Strategy 3: Data Structure Optimization

Use more efficient Redis data types:

```python
# BAD: Store JSON strings (bloated)
{
    'user:1': '{"name":"Alice","age":30,"email":"alice@ex.com","status":"active","tags":"vip,early-adopter","profile":"..."}'
}
# Size: 1-3KB per user

# BETTER: Use HASH (more compact)
redis_client.hset('user:1', mapping={
    'name': 'Alice',
    'age': 30,
    'email': 'alice@ex.com',
    'status': 'active'
})
# Size: 200-300B per user (5x smaller!)

# Retrieve
user = redis_client.hgetall('user:1')

# EVEN BETTER: Compress values
import json
import zlib

user_data = {'name': 'Alice', 'age': 30, ...}
compressed = zlib.compress(json.dumps(user_data))
redis_client.set('user:1:compressed', compressed)

# Decompress on read
decompressed = json.loads(zlib.decompress(redis_client.get('user:1:compressed')))
# Trade: CPU for memory (usually worth it)
```

**Data Type Sizes:**

| Type | Example | Size | Efficient? |
|------|---------|------|-----------|
| String | JSON {"data":"..."} | 1-5KB | No |
| String | Compressed JSON | 100-500B | Yes |
| Hash | {field1: val1, field2: val2} | 200B | Yes |
| Set | {"tag1", "tag2"} | 50B | Yes (tags) |
| Sorted Set | {member: score} | 100B | Yes (rankings) |
| List | [item1, item2] | 200B | Medium |

**Example: Session Optimization**

```python
# OLD: Store entire session as JSON string
session_json = json.dumps({
    'user_id': 123,
    'username': 'alice',
    'email': 'alice@ex.com',
    'roles': ['admin', 'user'],
    'preferences': {
        'theme': 'dark',
        'notifications': true,
        'timezone': 'UTC'
    },
    'login_time': '2026-04-06T10:00:00Z',
    'last_activity': '2026-04-06T11:00:00Z',
    'ip_address': '192.168.1.1',
    'user_agent': '...'
})
redis_client.setex('session:abc123', 86400, session_json)
# Size: 500B-1KB per session

# NEW: Use HASH for faster access + lower memory
redis_client.hset('session:abc123', mapping={
    'user_id': 123,
    'username': 'alice',
    'email': 'alice@ex.com',
    'roles': 'admin,user',  # String, not array
    'theme': 'dark',
    'login_time': '2026-04-06T10:00:00Z'
})
redis_client.expire('session:abc123', 86400)
# Size: 200-300B per session (2-3x savings!)

# Get specific fields (efficient)
username = redis_client.hget('session:abc123', 'username')
```

## Strategy 4: Compression

Compress large values:

```python
import zlib
import json

# Compress on write
def set_compressed(key, value, ttl=3600):
    compressed = zlib.compress(json.dumps(value).encode())
    redis_client.setex(key, ttl, compressed)

# Decompress on read
def get_compressed(key):
    data = redis_client.get(key)
    if data:
        return json.loads(zlib.decompress(data))
    return None

# Usage
large_data = {'report': [1000s of rows]}
set_compressed('report:2026-04', large_data)

# Retrieve
report = get_compressed('report:2026-04')

# Compression ratio: 10-20x for repeating data
# Tradeoff: CPU (compression) for memory (savings)
```

## Strategy 5: Memory-Efficient Hash Management

```python
# Avoid storing null/empty values
# BAD: Store every field
redis_client.hset('user:1', mapping={
    'name': 'Alice',
    'middle_name': '',      # Wastes 1 byte + overhead
    'phone': None,          # Wastes bytes + overhead
    'preferred_name': '',   # Wastes bytes
})

# GOOD: Only store non-empty
user = {'name': 'Alice'}
if email:
    user['email'] = email
redis_client.hset('user:1', mapping=user)
```

## Real Example: User Cache Optimization

```
BEFORE:
- 10M users
- Each stored as full JSON string
- Avg size: 3.2KB
- Total: 32GB

Breakdown:
- User ID: 10B
- Username: 30B
- Email: 50B
- Full profile: 1KB
- Preferences: 500B
- Activity log: 500B
- Other: 1KB

Strategy: Keep only hot data in Redis

AFTER:
- Only active users (30%): 3M users
- Essential fields only (username, email): 100B per
- Short TTL: 1 hour
- Compress large fields
- Total: 2GB (16x improvement!)

Implementation:
```

```python
class RedisUserCache:
    def __init__(self, redis_client, ttl=3600):
        self.redis = redis_client
        self.ttl = ttl  # 1 hour

    def set_user(self, user_id, username, email, avatar_url=None):
        # Store only essential fields
        user_key = f"user:{user_id}"
        mapping = {
            'username': username,
            'email': email,
        }
        if avatar_url:
            mapping['avatar'] = avatar_url

        self.redis.hset(user_key, mapping=mapping)
        self.redis.expire(user_key, self.ttl)

    def get_user(self, user_id):
        user_key = f"user:{user_id}"
        return self.redis.hgetall(user_key)

    def cache_stats(self):
        # Monitor what's cached
        info = self.redis.info('memory')
        return {
            'total_memory': info['used_memory_human'],
            'max_memory': self.redis.config_get('maxmemory'),
            'keys': self.redis.dbsize(),
            'avg_size': info['used_memory'] / max(self.redis.dbsize(), 1)
        }

# Usage
cache = RedisUserCache(redis_client, ttl=3600)
cache.set_user(1, 'alice', 'alice@ex.com', 'https://...')

stats = cache.cache_stats()
print(stats)  # Total memory, keys, avg size per key
```

## Monitoring Memory

```bash
# Check memory usage
redis-cli INFO memory

# Output:
# used_memory: 33GB
# used_memory_human: 33.45G
# used_memory_rss: 35GB (includes overhead)
# used_memory_peak: 40GB (peak usage)
# used_memory_dataset: 20GB (actual data)
# used_memory_overhead: 13GB (internal structures)

# Find keys using most memory
redis-cli --bigkeys
# Top 10 keys by size, helps identify bloat

# Memory by data type
redis-cli INFO stats
# Shows encoding info for memory efficiency

# Monitor in real-time
watch -n 1 'redis-cli INFO memory | grep used_memory'
```

## TTL Best Practices

```sql
-- Cache with reasonable TTL
SET user:cache:123 <json> EX 3600      # 1 hour

-- Session with longer TTL
SET session:abc <json> EX 86400        # 24 hours

-- Rate limit with exact TTL
SET ratelimit:ip:1.2.3.4 10 EX 60      # 1 minute (exact)

-- Background job with short TTL
SET job:xyz <data> EX 300              # 5 minutes (short)

-- Set default expiry on all keys
CONFIG SET notify-keyspace-events KA
# Subscribe to expiry events for cleanup
```

