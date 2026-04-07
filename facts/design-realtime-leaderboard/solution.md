# Real-Time Leaderboard Architecture

## The Problem

```sql
-- SQL approach (SLOW with 1M users)
SELECT user_id, score FROM leaderboard
ORDER BY score DESC
LIMIT 10;

-- This query:
-- ✗ Must scan all 1M rows
-- ✗ Must sort (O(n log n))
-- ✗ Takes 500ms+
-- ✗ Can't update real-time
-- ✗ Bottlenecks at scale
```

## Redis Sorted Sets Solution

Redis Sorted Sets are the perfect data structure for leaderboards:
- **O(1) insert/update**
- **O(log n) rank queries**
- **O(1) score by member**
- **In-memory (super fast)**
- **Automatically sorted**

```
Member → Score
user123 → 9500
user456 → 9200
user789 → 8900

Redis automatically maintains sorted order
Rank 1 = user123 (9500 points)
Rank 2 = user456 (9200 points)
Rank 3 = user789 (8900 points)
```

## Architecture

```
┌──────────────────────────────────┐
│   User Earns Points              │
│   Event: "Level Complete"        │
└────────────┬─────────────────────┘
             │
             ▼
┌──────────────────────────────────┐
│   Points Service                 │
│   - Calculate +100 points        │
│   - Update Redis (fast!)         │
│   - Async: Update database       │
└────────────┬─────────────────────┘
             │
             ▼
┌──────────────────────────────────┐
│   Redis Sorted Set               │
│   ZADD leaderboard 100 user123   │
│   (Rank updated instantly)       │
└────────────┬─────────────────────┘
             │
             ▼
┌──────────────────────────────────┐
│   Client Queries                 │
│   GET /leaderboard/top-10        │
│   Returns top 10 in < 5ms        │
└──────────────────────────────────┘
```

## Redis Commands

### Adding/Updating Score

```
ZADD leaderboard 1500 user123
ZADD leaderboard 2000 user456
ZADD leaderboard 1200 user789

# Update score (same command)
ZADD leaderboard 1700 user123  # user123 now has 1700 points
```

### Getting Rank

```
# Get rank (0-indexed)
ZREVRANK leaderboard user123
→ 0  (rank 1, highest score)

ZREVRANK leaderboard user789
→ 2  (rank 3, third highest)
```

### Getting Score

```
ZSCORE leaderboard user123
→ 1700
```

### Top N Players

```
# Get top 10 (with scores)
ZREVRANGE leaderboard 0 9 WITHSCORES
→ 1) user456
  2) 2000
  3) user123
  4) 1700
  5) user789
  6) 1200
  ...
```

### Range by Score

```
# Get all players with score 1500-2000
ZRANGEBYSCORE leaderboard 1500 2000
→ 1) user456 (2000)
  2) user123 (1700)
```

## Implementation

### Python

```python
import redis
from datetime import datetime

redis_client = redis.Redis(host='localhost', port=6379, db=0)

class LeaderboardService:
    LEADERBOARD_KEY = "leaderboard:global"
    MONTHLY_KEY_TEMPLATE = "leaderboard:monthly:{year}:{month}"

    def add_points(self, user_id: str, points: int):
        """Award points to user"""
        # Add to global leaderboard
        redis_client.zadd(self.LEADERBOARD_KEY, {user_id: points}, incr=True)

        # Add to monthly leaderboard
        now = datetime.now()
        monthly_key = self.MONTHLY_KEY_TEMPLATE.format(
            year=now.year,
            month=now.month
        )
        redis_client.zadd(monthly_key, {user_id: points}, incr=True)

    def get_rank(self, user_id: str) -> int:
        """Get user's current rank (1-indexed)"""
        rank = redis_client.zrevrank(self.LEADERBOARD_KEY, user_id)
        return rank + 1 if rank is not None else None

    def get_score(self, user_id: str) -> float:
        """Get user's current score"""
        return redis_client.zscore(self.LEADERBOARD_KEY, user_id)

    def get_top_players(self, limit: int = 10) -> list:
        """Get top N players"""
        results = redis_client.zrevrange(
            self.LEADERBOARD_KEY,
            0,
            limit - 1,
            withscores=True
        )

        return [
            {
                'rank': rank + 1,
                'user_id': user_id,
                'score': int(score)
            }
            for rank, (user_id, score) in enumerate(results)
        ]

    def get_player_context(self, user_id: str, context_size: int = 5) -> dict:
        """Get user's rank with surrounding players"""
        rank = redis_client.zrevrank(self.LEADERBOARD_KEY, user_id)

        if rank is None:
            return None

        # Get players around this user
        start = max(0, rank - context_size)
        end = rank + context_size

        context = redis_client.zrevrange(
            self.LEADERBOARD_KEY,
            start,
            end,
            withscores=True
        )

        return {
            'user_rank': rank + 1,
            'user_score': int(redis_client.zscore(self.LEADERBOARD_KEY, user_id)),
            'surrounding_players': [
                {
                    'rank': start + idx + 1,
                    'user_id': player_id,
                    'score': int(score),
                    'is_user': player_id == user_id
                }
                for idx, (player_id, score) in enumerate(context)
            ]
        }

    def get_monthly_leaderboard(self, year: int, month: int, limit: int = 10):
        """Get leaderboard for specific month"""
        key = self.MONTHLY_KEY_TEMPLATE.format(year=year, month=month)
        results = redis_client.zrevrange(key, 0, limit - 1, withscores=True)

        return [
            {
                'rank': rank + 1,
                'user_id': user_id,
                'score': int(score)
            }
            for rank, (user_id, score) in enumerate(results)
        ]

    def get_percentile(self, user_id: str) -> float:
        """Get user's percentile rank"""
        rank = redis_client.zrevrank(self.LEADERBOARD_KEY, user_id)
        total = redis_client.zcard(self.LEADERBOARD_KEY)

        if rank is None or total == 0:
            return 0

        percentile = ((total - rank) / total) * 100
        return round(percentile, 2)
```

### FastAPI Endpoint

```python
from fastapi import FastAPI, Query

app = FastAPI()
leaderboard = LeaderboardService()

@app.post("/points")
def add_points(user_id: str, points: int):
    """Award points to user"""
    leaderboard.add_points(user_id, points)
    return {
        'user_id': user_id,
        'points_added': points,
        'new_rank': leaderboard.get_rank(user_id),
        'total_score': leaderboard.get_score(user_id)
    }

@app.get("/leaderboard/top")
def get_top_leaderboard(limit: int = Query(10, ge=1, le=100)):
    """Get top N players"""
    return leaderboard.get_top_players(limit)

@app.get("/leaderboard/user/{user_id}")
def get_user_leaderboard(user_id: str):
    """Get user's rank and surrounding context"""
    context = leaderboard.get_player_context(user_id, context_size=5)
    if not context:
        return {"error": "User not found"}, 404
    return context

@app.get("/leaderboard/monthly")
def get_monthly_leaderboard(year: int, month: int, limit: int = Query(10)):
    """Get leaderboard for specific month"""
    return leaderboard.get_monthly_leaderboard(year, month, limit)

@app.get("/user/{user_id}/stats")
def get_user_stats(user_id: str):
    """Get user's stats"""
    rank = leaderboard.get_rank(user_id)
    if not rank:
        return {"error": "User not found"}, 404

    return {
        'user_id': user_id,
        'rank': rank,
        'score': int(leaderboard.get_score(user_id)),
        'percentile': leaderboard.get_percentile(user_id)
    }
```

## Performance Comparison

| Operation | SQL Database | Redis Sorted Set |
|-----------|--------------|------------------|
| Add/update score | 10-50ms | <1ms |
| Get top 10 | 100-500ms | <5ms |
| Get user rank | 100-500ms | <5ms |
| Get percentile | 1000ms+ | <10ms |
| Concurrent users | Degrades | No degradation |

## Scaling Strategies

### 1. Multiple Leaderboards

```python
# Different leaderboards for different purposes
leaderboards = {
    'global': 'leaderboard:global',
    'monthly': f'leaderboard:monthly:{year}:{month}',
    'weekly': f'leaderboard:weekly:w{week_number}',
    'by_country': f'leaderboard:country:{country}',
    'by_game_mode': f'leaderboard:gamemode:{mode}'
}

# User might rank #50 globally but #1 in their country
```

### 2. Leaderboard Snapshots

```python
# For very large leaderboards (10M+ users)
# Take daily snapshots to reduce memory

# Original leaderboard: 10M users in Redis
# Daily snapshot: Top 10K only
# Archive to database for analytics
```

### 3. Tiered Leaderboards

```
Top 100 → Tier 1 (in Redis)
Top 1K → Tier 2 (in Redis)
Top 10K → Tier 3 (in Redis)
Tail → Database only (lower tier users don't query)
```

## Persistence

```python
# Redis is in-memory, persist to database for redundancy

def persist_leaderboard():
    """Backup Redis to database periodically"""
    results = redis_client.zrevrange(
        'leaderboard:global',
        0,
        -1,  # All entries
        withscores=True
    )

    for rank, (user_id, score) in enumerate(results):
        db.leaderboard.update_one(
            {'user_id': user_id},
            {
                '$set': {
                    'score': int(score),
                    'rank': rank + 1,
                    'last_updated': datetime.utcnow()
                }
            },
            upsert=True
        )

# Run every hour
scheduler.add_job(persist_leaderboard, 'interval', hours=1)
```

## Memory Optimization

```
1M users × 50 bytes per entry ≈ 50MB Redis
10M users × 50 bytes ≈ 500MB Redis

Acceptable for most deployments. If too large:
├─ Keep only top 100K in Redis
├─ Full list in database for queries
├─ Sync periodically
```

## Real-World Example: Game Leaderboard

```python
@app.post("/game/complete/{game_id}")
def complete_game(game_id: str, user_id: str, score: int, duration_sec: int):
    """User completes game, update leaderboard"""

    # Calculate bonus points
    bonus = 0
    if duration_sec < 60:
        bonus = 100  # Speed bonus

    total_points = score + bonus

    # Update leaderboards
    leaderboard.add_points(user_id, total_points)

    # Get new stats
    new_rank = leaderboard.get_rank(user_id)
    top_players = leaderboard.get_top_players(10)

    return {
        'points_earned': total_points,
        'new_rank': new_rank,
        'top_10': top_players
    }
```

Redis sorted sets are the industry standard for leaderboards at scale.
