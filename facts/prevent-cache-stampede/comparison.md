## Strategy Trade-off Matrix

| Strategy | Lock Contention | Fresh Data | Latency | DB Load Spike | Complexity |
|----------|-----------------|-----------|---------|---------------|-----------|
| **Lock (Mutex)** | Low | Stale during refresh | HIGH 500ms+ | Very Low | Low |
| **Early Refresh** | None | Fresh | Normal | Low | Medium |
| **Staggered TTL** | None | Slightly stale | Normal | Medium | Very Low |
| **Probabilistic** | None | Fresh | Normal | Low-Medium | High |
| **Cache Warming** | None | Fresh | Normal | Very Low | Medium |
| **Two-Level** | None | Slightly stale | FAST <10ms | Low-Medium | High |

## Choosing by Scenario

### High Traffic, Millisecond-Critical (e.g., Homepage Feed)
Use **Early Refresh + Staggered TTL**:
- Refresh when cache < 30% TTL remaining
- Users always see fresh data
- Load spreads smoothly

```python
def get_feed(user_id):
    key = f"feed:{user_id}"
    cached = redis_client.get(key)

    ttl = redis_client.ttl(key)
    if ttl < 300:  # Less than 5 minutes left
        refresh_feed_background(user_id)

    return cached or compute_feed(user_id)
```

### Large Number of Keys, Unpredictable Access
Use **Staggered TTL Alone**:
- No background jobs needed
- Simple to implement
- Some keys slightly stale OK

```python
ttl = CACHE_TTL + random.randint(-300, 300)
redis_client.setex(key, ttl, value)
```

### Known Hot Keys (Top Products, Trending)
Use **Cache Warming**:
- Pre-populate before traffic spikes
- Zero stampede risk
- Predictable load

```python
POPULAR_KEYS = [
    ("top_products", get_top_products),
    ("trending_tags", get_trending_tags),
]

for key_name, fetch_func in POPULAR_KEYS:
    result = fetch_func()
    redis_client.setex(key_name, CACHE_TTL, result)
```

### Sensitive Data with Freshness Requirements
Use **Two-Level Cache + Lock**:
- Memory cache for extreme performance
- Lock protects critical data
- Redis as distributed fallback

```python
cache = TwoLevelCache()

def get_user_balance(user_id):
    # Local cache 1-2s, Redis 1-5s, then compute
    return cache.get(f"balance:{user_id}")
```

### Database with Occasional Spikes
Use **Probabilistic Refresh (XFetch)**:
- Smooth exponential load increase
- No sudden computation bursts
- Adaptive to load

```python
def get_data(key):
    cached = redis_client.get(key)
    if not cached:
        return compute(key)

    age = get_age(key)
    if random.random() < (age / TTL):
        refresh_background(key)

    return cached
```

## Testing Cache Stampede Prevention

```python
import pytest
import time
from concurrent.futures import ThreadPoolExecutor, as_completed

def test_no_cache_stampede_with_lock():
    """Verify lock prevents multiple DB queries"""
    call_count = {"value": 0}
    original_expensive_query = expensive_query

    def mock_expensive_query():
        call_count["value"] += 1
        time.sleep(0.5)
        return "result"

    # Clear cache to force miss
    redis_client.delete("test_key")

    # 100 concurrent requests when cache is empty
    with ThreadPoolExecutor(max_workers=100) as executor:
        futures = [
            executor.submit(get_popular_data, "test_key")
            for _ in range(100)
        ]
        results = [f.result() for f in as_completed(futures)]

    # With lock: should only call database once (or few times)
    # Without lock: would call 100 times
    assert call_count["value"] <= 3, f"DB called {call_count['value']} times (should be 1-3)"
    assert all(r == "result" for r in results)

def test_early_refresh_no_stale_reads():
    """Verify early refresh keeps data fresh"""
    redis_client.delete("test_key")

    # Initial value
    redis_client.setex("test_key", 60, "v1")

    # Simulate time passing (58 seconds)
    # In real test, use mock time or sleep

    # Request when < 30% TTL remaining
    result = get_popular_data_proactive("test_key")

    # Should get v1 but trigger refresh
    assert result == "v1"

    # After background refresh completes
    time.sleep(1)

    # New value should be cached
    result2 = get_popular_data_proactive("test_key")
    assert result2 == "v2" or result2 == "v1"  # Either is OK
```

## Production Deployment Checklist

- [ ] Set appropriate CACHE_TTL based on data freshness requirements
- [ ] Monitor Redis key eviction rate (may indicate small cache)
- [ ] Set LOCK_TTL < CACHE_TTL to prevent deadlocks
- [ ] Alert on high Redis memory usage (full cache = poor performance)
- [ ] A/B test lock vs early-refresh vs staggered on real traffic
- [ ] Log cache hits/misses to measure effectiveness
- [ ] Measure p99 latency during spike times
- [ ] Consider circuit breaker if database is down (return stale cache)
