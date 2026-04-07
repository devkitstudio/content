## Middleware for Automatic Read Routing

```python
# Django/Flask middleware to route reads automatically

from functools import wraps
from django.db import connections
from contextlib import contextmanager
import json
from datetime import datetime, timedelta

class ReplicationLagRouter:
    def __init__(self, max_lag_ms=1000):
        self.max_lag_ms = max_lag_ms
        self.max_retries = 5
        self.retry_delay_ms = 100

    def get_lag_bytes(self):
        """Get replication lag in bytes"""
        primary_lsn = connections['primary'].cursor().execute(
            "SELECT pg_current_wal_lsn()::text"
        ).fetchone()[0]

        replica_lsn = connections['replica'].cursor().execute(
            "SELECT pg_last_wal_replay_lsn()::text"
        ).fetchone()[0]

        # Convert LSN (e.g., "0/12345678") to bytes
        def lsn_to_bytes(lsn_str):
            parts = lsn_str.split('/')
            return (int(parts[0], 16) << 32) + int(parts[1], 16)

        return lsn_to_bytes(primary_lsn) - lsn_to_bytes(replica_lsn)

    def estimate_lag_ms(self):
        """Estimate lag in milliseconds (rough: 10MB/s replication)"""
        lag_bytes = self.get_lag_bytes()
        # 10 MB/s replication rate
        lag_seconds = lag_bytes / (10 * 1024 * 1024)
        return lag_seconds * 1000

    def read_from_primary_if_lagged(self, func):
        """Decorator: use primary if replica lag exceeds threshold"""
        @wraps(func)
        def wrapper(*args, **kwargs):
            lag_ms = self.estimate_lag_ms()

            if lag_ms > self.max_lag_ms:
                # Use primary
                with connections['primary'].cursor() as cursor:
                    # Temporarily override the default connection
                    return func(*args, **kwargs)
            else:
                # Use replica
                with connections['replica'].cursor() as cursor:
                    return func(*args, **kwargs)

        return wrapper

# Usage in Django view:
router = ReplicationLagRouter(max_lag_ms=500)

@router.read_from_primary_if_lagged
def get_user_profile(user_id):
    return User.objects.get(id=user_id)
```

## FastAPI Implementation with Context Tracking

```python
from fastapi import FastAPI, Request
from contextvars import ContextVar
from datetime import datetime, timedelta
import asyncpg
import time

app = FastAPI()

# Context variables to track writes
write_timestamp: ContextVar[datetime] = ContextVar('write_timestamp', default=None)
write_lsn: ContextVar[str] = ContextVar('write_lsn', default=None)

class DatabasePool:
    def __init__(self, primary_url: str, replica_url: str):
        self.primary_url = primary_url
        self.replica_url = replica_url
        self.primary_pool = None
        self.replica_pool = None

    async def init(self):
        self.primary_pool = await asyncpg.create_pool(self.primary_url)
        self.replica_pool = await asyncpg.create_pool(self.replica_url)

    async def write(self, query: str, *args):
        """Execute write on primary, track position"""
        async with self.primary_pool.acquire() as conn:
            result = await conn.execute(query, *args)

            # Get current WAL position
            lsn = await conn.fetchval("SELECT pg_current_wal_lsn()::text")
            write_lsn.set(lsn)
            write_timestamp.set(datetime.utcnow())

            return result

    async def read(self, query: str, *args):
        """Execute read on replica if safe, otherwise primary"""
        last_lsn = write_lsn.get()
        last_write = write_timestamp.get()

        # If wrote < 1 second ago, use primary
        if last_write and (datetime.utcnow() - last_write) < timedelta(seconds=1):
            async with self.primary_pool.acquire() as conn:
                return await conn.fetch(query, *args)

        # If have LSN, wait for replica to catch up
        if last_lsn:
            for attempt in range(10):
                async with self.replica_pool.acquire() as conn:
                    replica_lsn = await conn.fetchval(
                        "SELECT pg_last_wal_replay_lsn()::text"
                    )

                if replica_lsn >= last_lsn:
                    # Replica caught up, safe to read
                    async with self.replica_pool.acquire() as conn:
                        return await conn.fetch(query, *args)

                await asyncio.sleep(0.05)

        # Replica lagging or no tracking, fall back to primary
        async with self.primary_pool.acquire() as conn:
            return await conn.fetch(query, *args)

db = DatabasePool(
    primary_url="postgresql://user@primary:5432/db",
    replica_url="postgresql://user@replica:5432/db"
)

@app.on_event("startup")
async def startup():
    await db.init()

@app.post("/users/{user_id}")
async def update_user(user_id: int, data: dict):
    """Write to primary"""
    await db.write(
        "UPDATE users SET name = $1 WHERE id = $2",
        data['name'], user_id
    )
    return {"status": "updated"}

@app.get("/users/{user_id}")
async def get_user(user_id: int):
    """Read from replica (with fallback)"""
    rows = await db.read(
        "SELECT * FROM users WHERE id = $1",
        user_id
    )
    return rows[0] if rows else None
```

## PostgreSQL Configuration

```sql
-- Primary: streaming replication configuration
-- postgresql.conf

wal_level = replica
max_wal_senders = 10
wal_keep_size = 1GB
hot_standby = on

-- Enable replication slot for reliable delivery
CREATE_REPLICATION_SLOT(slot_name := 'replica_slot', decoding := 'test_decoding')

-- Replica: connection configuration
primary_conninfo = 'host=primary port=5432 user=replication password=password'
recovery_target_timeline = 'latest'

-- Monitor replication status
SELECT slot_name, slot_type, active, restart_lsn
FROM pg_replication_slots;

SELECT client_addr, state, sync_state, write_lag, flush_lag, replay_lag
FROM pg_stat_replication;
```

## Testing Replication Lag Handling

```python
import pytest
import asyncio
from unittest.mock import AsyncMock, patch

@pytest.mark.asyncio
async def test_read_from_replica_when_no_lag():
    db = DatabasePool(primary_url="...", replica_url="...")
    await db.init()

    # Simulate no recent writes
    write_timestamp.set(None)

    with patch.object(db.replica_pool, 'acquire') as mock_replica:
        await db.read("SELECT * FROM users WHERE id = $1", 42)
        mock_replica.assert_called_once()

@pytest.mark.asyncio
async def test_read_from_primary_after_write():
    db = DatabasePool(primary_url="...", replica_url="...")
    await db.init()

    # Execute write
    await db.write("UPDATE users SET name = $1 WHERE id = $2", "Alice", 42)

    # Immediately read should use primary
    with patch.object(db.primary_pool, 'acquire') as mock_primary:
        await db.read("SELECT * FROM users WHERE id = $1", 42)
        mock_primary.assert_called_once()
```

## Monitoring and Alerting

```python
async def monitor_replication_lag():
    """Continuously monitor replica lag, alert if excessive"""
    while True:
        lag_ms = db.estimate_lag_ms()

        # Alert if lag > 5 seconds
        if lag_ms > 5000:
            logger.error(f"High replication lag: {lag_ms}ms")
            # Send to monitoring service (Datadog, New Relic, etc.)
            send_alert(
                metric="database.replication_lag_ms",
                value=lag_ms,
                severity="high"
            )

        await asyncio.sleep(5)  # Check every 5 seconds
```
