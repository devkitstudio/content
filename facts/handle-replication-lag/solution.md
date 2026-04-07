## Strategies for Handling Replication Lag

### Strategy 1: Read-From-Primary for Recent Writes

**Problem:**
```
User writes profile → Replication lag 500ms → Read returns old data
```

**Solution: Track written data**
```python
# Application layer
from datetime import datetime, timedelta

class User:
    def __init__(self, user_id):
        self.user_id = user_id
        self.last_write_time = None

    def update_profile(self, name, email):
        # Write to primary
        db_primary.execute(
            "UPDATE users SET name = %s, email = %s WHERE id = %s",
            (name, email, self.user_id)
        )
        # Track write time
        self.last_write_time = datetime.utcnow()

    def get_profile(self):
        # If written < 1 second ago, read from primary
        if self.last_write_time and (datetime.utcnow() - self.last_write_time) < timedelta(seconds=1):
            return db_primary.query("SELECT * FROM users WHERE id = %s", self.user_id)

        # Otherwise safe to read from replica
        return db_replica.query("SELECT * FROM users WHERE id = %s", self.user_id)
```

### Strategy 2: Causal Consistency with Version Tracking

**Use LSN (Log Sequence Number) to guarantee read-after-write:**

```python
# After write, get the primary's current LSN
def write_and_track(user_id, name):
    # Write to primary
    db_primary.execute(
        "UPDATE users SET name = %s WHERE id = %s",
        (name, user_id)
    )

    # Get primary's current LSN (replication position)
    lsn = db_primary.query_scalar("SELECT pg_current_wal_lsn()")

    # Store in session/cache
    return {"lsn": lsn, "user_id": user_id}

def read_with_causality(user_id, lsn):
    # Wait for replica to catch up to this LSN
    max_retries = 10
    for attempt in range(max_retries):
        replica_lsn = db_replica.query_scalar(
            "SELECT pg_last_wal_replay_lsn()"
        )

        if replica_lsn >= lsn:
            # Replica has replicated the write, safe to read
            return db_replica.query(
                "SELECT * FROM users WHERE id = %s",
                user_id
            )

        # Wait and retry
        time.sleep(0.1)

    # Timeout, fall back to primary
    return db_primary.query(
        "SELECT * FROM users WHERE id = %s",
        user_id
    )
```

### Strategy 3: Dedicated Primary for User's Own Data

```python
# User only reads their own data from primary (fast writes visible immediately)
class UserRepository:
    def get_user_profile(self, user_id, is_own_profile=False):
        if is_own_profile:
            # Read from primary (user's own data is authoritative)
            return db_primary.query("SELECT * FROM users WHERE id = %s", user_id)
        else:
            # Read from replica (public profile, eventual consistency OK)
            return db_replica.query("SELECT * FROM users WHERE id = %s", user_id)
```

### Strategy 4: Lag Monitoring & Failover

```python
def get_replica_lag_ms():
    """Check replication lag in milliseconds"""
    primary_lsn = db_primary.query_scalar("SELECT pg_current_wal_lsn()::text")
    replica_lsn = db_replica.query_scalar("SELECT pg_last_wal_replay_lsn()::text")

    lag_bytes = lsn_to_bytes(primary_lsn) - lsn_to_bytes(replica_lsn)

    # Rough: 10MB/s replication speed
    lag_ms = (lag_bytes / (10 * 1024 * 1024)) * 1000
    return lag_ms

def query_with_fallback(sql, max_lag_ms=1000):
    """Route read, fallback if replica too far behind"""
    lag_ms = get_replica_lag_ms()

    if lag_ms > max_lag_ms:
        # Too much lag, use primary
        return db_primary.query(sql)
    else:
        # Replica is current enough
        return db_replica.query(sql)
```

### Strategy 5: Sticky Writes

```python
# All operations on a resource go to the same server temporarily

class Session:
    def __init__(self, user_id):
        self.user_id = user_id
        self.sticky_primary = False

    def execute_write(self, sql):
        db_primary.execute(sql)
        self.sticky_primary = True  # Next reads from primary

    def execute_read(self, sql):
        if self.sticky_primary:
            # User just wrote, use primary for next ~5 seconds
            return db_primary.query(sql)
        else:
            # Normal operation, use replica
            return db_replica.query(sql)

# Example:
session = Session(user_id=42)
session.execute_write("UPDATE users SET email = 'new@example.com' WHERE id = 42")
# Next 5 seconds, reads go to primary
user = session.execute_read("SELECT * FROM users WHERE id = 42")  # From primary
```

## When to Use Each Strategy

| Strategy | Pros | Cons | Use Case |
|----------|------|------|----------|
| **Read-From-Primary** | Simple, reliable | High load on primary | Small datasets, critical reads |
| **Version Tracking** | Precise, efficient | Complex | Web apps, social media |
| **Own Data → Primary** | User-friendly | Splits traffic | User profiles, dashboards |
| **Lag Monitoring** | Adaptive | Monitoring overhead | Variable lag, high availability |
| **Sticky Writes** | Predictable | Uneven distribution | Shopping carts, forms |
