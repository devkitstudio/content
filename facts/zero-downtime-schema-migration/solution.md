## The Problem: Why ADD COLUMN Locks the Table

```sql
-- BLOCKS ALL READS/WRITES for 10+ minutes on 50M rows
ALTER TABLE users ADD COLUMN verified_at TIMESTAMP NOT NULL DEFAULT now();
```

This locks because PostgreSQL must:
1. Rewrite entire table to add column to every row
2. Create new index entries
3. Update all table metadata

## Safe Multi-Step Migration Pattern

### Step 1: Add Column as NULL (Fast, non-blocking)

```sql
-- Takes <100ms even on 50M rows
ALTER TABLE users ADD COLUMN verified_at TIMESTAMP;
```

**Why:** No DEFAULT forces rewrite. Column is NULL for existing rows.

### Step 2: Build Index (Concurrent, in background)

```sql
-- Creates index WITHOUT locking table for writes
CREATE INDEX CONCURRENTLY idx_users_verified_at
ON users(verified_at);
-- Takes minutes but doesn't block app
```

### Step 3: Add Default for Future Rows

```sql
-- Locks briefly (< 1 second), only metadata change
ALTER TABLE users ALTER COLUMN verified_at SET DEFAULT now();
```

### Step 4: Backfill Existing Rows (Parallel chunks)

```sql
-- Run in background, safe for production
UPDATE users SET verified_at = created_at
WHERE verified_at IS NULL
LIMIT 100000;  -- Update in batches

-- In loop: sleep 1 second, repeat until no more rows
```

### Step 5: Add NOT NULL Constraint

PostgreSQL 11+: Skip locking with NOT VALID + validation:

```sql
-- Step A: Add constraint without checking existing rows (instant)
ALTER TABLE users ADD CONSTRAINT users_verified_at_not_null
CHECK (verified_at IS NOT NULL) NOT VALID;

-- Step B: Backfill remaining NULLs
UPDATE users SET verified_at = created_at WHERE verified_at IS NULL;

-- Step C: Validate constraint (locks briefly on 50M rows)
ALTER TABLE users VALIDATE CONSTRAINT users_verified_at_not_null;

-- Step D: Convert to proper NOT NULL
ALTER TABLE users DROP CONSTRAINT users_verified_at_not_null;
ALTER TABLE users ALTER COLUMN verified_at SET NOT NULL;
```

Or PostgreSQL 12+ simpler approach:

```sql
-- Single command, handles everything without full rewrite
ALTER TABLE users ADD COLUMN verified_at TIMESTAMP NOT NULL DEFAULT now();
-- Uses deferred default: stored once, applied logically per row
```

## Real-World Example: Django Migration

```python
# migrations/0001_add_verified_at.py

from django.db import migrations, models

class Migration(migrations.Migration):
    dependencies = [
        ('users', '0000_previous'),
    ]

    operations = [
        # Step 1: Add nullable column
        migrations.AddField(
            model_name='user',
            name='verified_at',
            field=models.DateTimeField(null=True, blank=True),
        ),

        # Step 2: Create index concurrently
        migrations.RunSQL(
            'CREATE INDEX CONCURRENTLY idx_users_verified_at ON users(verified_at);',
            reverse_sql='DROP INDEX IF EXISTS idx_users_verified_at;',
        ),

        # Step 3: Add default
        migrations.AlterField(
            model_name='user',
            name='verified_at',
            field=models.DateTimeField(
                null=True,
                blank=True,
                default=django.utils.timezone.now
            ),
        ),
    ]
```

Then in separate data migration:

```python
# migrations/0002_backfill_verified_at.py

def backfill_verified_at(apps, schema_editor):
    User = apps.get_model('users', 'User')
    batch_size = 10000

    while True:
        users = User.objects.filter(verified_at__isnull=True)[:batch_size]
        if not users.exists():
            break

        for user in users:
            user.verified_at = user.created_at

        User.objects.bulk_update(users, ['verified_at'], batch_size=1000)
        time.sleep(1)  # Give DB a break

class Migration(migrations.Migration):
    dependencies = [
        ('users', '0001_add_verified_at'),
    ]

    operations = [
        migrations.RunPython(backfill_verified_at),
    ]
```

Finally, add constraint in separate migration:

```python
# migrations/0003_make_verified_at_required.py

class Migration(migrations.Migration):
    operations = [
        migrations.AlterField(
            model_name='user',
            name='verified_at',
            field=models.DateTimeField(),  # Remove null=True
        ),
    ]
```

## Monitoring During Migration

```sql
-- Check progress of backfill
SELECT COUNT(*) as remaining_null
FROM users
WHERE verified_at IS NULL;

-- Check index build progress (PostgreSQL 9.6+)
SELECT schemaname, tablename, indexname, idx_scan, idx_tup_read, idx_tup_fetch
FROM pg_stat_user_indexes
WHERE indexname = 'idx_users_verified_at';

-- Kill long-running migration if needed
SELECT pid, usename, query, query_start, state
FROM pg_stat_activity
WHERE query LIKE '%ALTER TABLE%';
-- SELECT pg_terminate_backend(pid_number);
```
