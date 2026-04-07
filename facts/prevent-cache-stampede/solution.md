## The Problem: Cache Stampede

```
Key expires at 12:00:00
10,000 requests at 12:00:01 all find cache miss
10,000 requests hit database simultaneously (resource exhausted)
```

## Strategy 1: Distributed Lock (Mutex)

Only one request computes, others wait for result:

```python
import redis
import time
from datetime import timedelta

redis_client = redis.Redis()
CACHE_TTL = 3600
LOCK_TTL = 10

def get_popular_data(key):
    # Try to read from cache
    cached = redis_client.get(key)
    if cached:
        return cached

    # Cache miss: try to acquire lock
    lock_key = f"lock:{key}"
    lock_acquired = redis_client.set(
        lock_key,
        "1",
        nx=True,  # Only set if not exists
        ex=LOCK_TTL  # Auto-release after 10s
    )

    if lock_acquired:
        # We have the lock, compute and cache
        try:
            result = expensive_query()  # 500ms database call
            redis_client.setex(key, CACHE_TTL, result)
            return result
        finally:
            redis_client.delete(lock_key)
    else:
        # Someone else has lock, wait and retry
        for _ in range(50):  # Wait up to 5 seconds
            time.sleep(0.1)
            cached = redis_client.get(key)
            if cached:
                return cached

        # Timeout waiting, compute ourselves (fallback)
        return expensive_query()
```

**Pros:** Prevents multiple concurrent computes
**Cons:** Increased latency for waiting requests (500ms+ computation)

## Strategy 2: Early Refresh (Proactive)

Refresh cache BEFORE it expires:

```python
def get_popular_data_proactive(key):
    cached = redis_client.get(key)

    # Check if expiring soon (< 30% of TTL left)
    ttl = redis_client.ttl(key)
    if ttl == -2:  # Key expired
        # Compute fresh
        result = expensive_query()
        redis_client.setex(key, CACHE_TTL, result)
        return result
    elif ttl > 0 and ttl < (CACHE_TTL * 0.3):  # Less than 30% TTL remaining
        # Refresh in background
        refresh_cache_async(key)  # Non-blocking
        return cached  # Serve stale data while refreshing

    return cached

def refresh_cache_async(key):
    """Background task to refresh cache"""
    try:
        result = expensive_query()
        redis_client.setex(key, CACHE_TTL, result)
    except Exception as e:
        logger.error(f"Cache refresh failed for {key}: {e}")
```

**Pros:** Never serves stale data to users
**Cons:** Requires background job infrastructure

## Strategy 3: Staggered TTL (Random Expiration)

Vary TTL so keys don't expire at same time:

```python
import random

CACHE_TTL = 3600
TTL_VARIANCE = 300  # +/- 5 minutes

def get_popular_data_staggered(key):
    cached = redis_client.get(key)
    if cached:
        return cached

    # Compute and cache with random TTL
    result = expensive_query()

    # TTL between 3300-3900 seconds (3600 +/- 300)
    ttl = CACHE_TTL + random.randint(-TTL_VARIANCE, TTL_VARIANCE)
    redis_client.setex(key, ttl, result)

    return result
```

**Pros:** Simple, spreads load over time
**Cons:** Some keys expire earlier, slight freshness variance

## Strategy 4: Probabilistic Early Expiration (XFetch)

Refresh probabilistically based on age:

```python
import math
import time

def get_popular_data_xfetch(key):
    result = redis_client.get(key)

    if result is None:
        # Cache miss, compute
        return expensive_query()

    # Cache hit: check if we should early-refresh
    # Probability increases as cache ages
    age = time.time() - float(redis_client.get(f"{key}:age"))
    ttl = CACHE_TTL

    # Refresh probability: 0% at start, 100% at expiration
    refresh_probability = age / ttl

    if random.random() < refresh_probability:
        # Refresh in background (don't block user)
        refresh_cache_async(key)

    return result
```

**Pros:** Smooth load distribution, no sudden spikes
**Cons:** More complex logic

## Strategy 5: Cache Warming

Pre-populate cache before spikes:

```python
import schedule
import time

def warm_cache_periodically():
    """Run every 30 minutes to refresh popular keys"""
    popular_keys = [
        "top_products:homepage",
        "user_counts:dashboard",
        "trending_feeds:social",
    ]

    for key in popular_keys:
        try:
            result = expensive_query()
            redis_client.setex(key, CACHE_TTL, result)
        except Exception as e:
            logger.error(f"Cache warming failed for {key}: {e}")

# Schedule the warming
schedule.every(30).minutes.do(warm_cache_periodically)

# Or trigger after predictable events
@app.post("/admin/clear-cache")
def admin_clear_cache():
    for key in popular_keys:
        warm_cache_periodically()
    return {"status": "cache warmed"}
```

**Pros:** No stampede, minimal computation during spikes
**Cons:** Requires predicting popular items

## Strategy 6: Two-Level Cache (Memory + Redis)

```python
from functools import lru_cache
import threading

class TwoLevelCache:
    def __init__(self):
        self.local_cache = {}
        self.local_ttl = {}
        self.lock = threading.RLock()

    def get(self, key):
        # Level 1: Process memory (instant, 1-2 seconds)
        with self.lock:
            if key in self.local_cache:
                if time.time() < self.local_ttl[key]:
                    return self.local_cache[key]
                else:
                    del self.local_cache[key]
                    del self.local_ttl[key]

        # Level 2: Redis (fast, few milliseconds)
        redis_val = redis_client.get(key)
        if redis_val:
            with self.lock:
                self.local_cache[key] = redis_val
                self.local_ttl[key] = time.time() + 5  # Keep locally for 5s
            return redis_val

        # Level 3: Compute (expensive)
        result = expensive_query()

        # Populate both levels
        redis_client.setex(key, CACHE_TTL, result)
        with self.lock:
            self.local_cache[key] = result
            self.local_ttl[key] = time.time() + 5

        return result

cache = TwoLevelCache()

def get_popular_data(key):
    return cache.get(key)
```

**Pros:** Multiple fallback layers, very fast
**Cons:** Memory overhead, complexity
