# Preventing Overlapping Cron Job Executions

## Problem: Overlapping Executions

```
Timeline (24-hour):
00:00 - Cron starts (takes 2 hours)
01:00 - Next cron starts (first still running!)
02:00 - First cron finishes
03:00 - Next cron starts (second still running)
...

Result: Multiple copies running simultaneously, duplicate work
```

## Strategy 1: Redis Distributed Lock

Most common approach for multi-server environments.

```javascript
const cron = require('node-cron');
const redis = require('redis');
const redisClient = redis.createClient();

// Schedule job to run daily at midnight
cron.schedule('0 0 * * *', async () => {
  const lockKey = 'cron-job:daily-report';
  const lockValue = Date.now().toString();
  const lockTTL = 3600 * 2; // 2 hours (longer than max job duration)

  try {
    // Try to acquire lock
    const acquired = await redisClient.set(
      lockKey,
      lockValue,
      'NX', // Only set if not exists
      'EX', // Set expiration
      lockTTL
    );

    if (!acquired) {
      console.log('Cron job already running, skipping');
      return;
    }

    console.log('Lock acquired, starting job');

    // Do the work
    await generateDailyReport();

    console.log('Job completed successfully');
  } catch (err) {
    console.error('Cron job error:', err);
  } finally {
    // Release lock only if we still own it
    const currentLock = await redisClient.get(lockKey);
    if (currentLock === lockValue) {
      await redisClient.del(lockKey);
      console.log('Lock released');
    }
  }
});

async function generateDailyReport() {
  // Simulate long-running task
  console.log('Starting report generation...');
  const startTime = Date.now();

  // Do actual work here
  await new Promise(resolve => setTimeout(resolve, 7200000)); // 2 hours

  console.log(`Report generation took ${Date.now() - startTime}ms`);
}
```

**Pros:**
- Works across multiple servers
- Automatic lock expiration
- Simple implementation

**Cons:**
- Requires Redis
- Lock TTL must be > job duration
- Clock skew can cause issues

---

## Strategy 2: Database Lock (PostgreSQL)

For systems without Redis, use database advisory locks.

```javascript
const cron = require('node-cron');
const pool = require('./database-pool');

cron.schedule('0 0 * * *', async () => {
  const client = await pool.connect();

  try {
    // Try to acquire advisory lock (lock ID 12345)
    const lockQuery = `SELECT pg_try_advisory_lock(12345) as acquired`;
    const result = await client.query(lockQuery);
    const lockAcquired = result.rows[0].acquired;

    if (!lockAcquired) {
      console.log('Lock not acquired, job already running');
      return;
    }

    console.log('Lock acquired, starting job');

    try {
      // Do the work
      await generateDailyReport(client);

      console.log('Job completed successfully');
    } finally {
      // Release lock
      await client.query(`SELECT pg_advisory_unlock(12345)`);
    }
  } catch (err) {
    console.error('Cron error:', err);
  } finally {
    client.release();
  }
});

async function generateDailyReport(client) {
  // Use same connection for all queries
  const yesterday = new Date();
  yesterday.setDate(yesterday.getDate() - 1);

  const query = `
    SELECT * FROM transactions
    WHERE DATE(created_at) = $1
  `;

  const results = await client.query(query, [yesterday]);
  console.log(`Processing ${results.rows.length} transactions`);

  // Insert summary
  await client.query(`
    INSERT INTO daily_reports (report_date, transaction_count, total_amount)
    VALUES ($1, $2, $3)
  `, [yesterday, results.rows.length, calculateTotal(results.rows)]);
}
```

**Pros:**
- Built-in database feature
- Very reliable
- Works without external services

**Cons:**
- Only PostgreSQL (advisory locks)
- Requires database connection
- Lock is released on connection close

---

## Strategy 3: File-Based Lock

Simplest for single-server deployments.

```javascript
const cron = require('node-cron');
const fs = require('fs/promises');
const path = require('path');

const lockFilePath = path.join('/tmp', 'cron-job.lock');

cron.schedule('0 0 * * *', async () => {
  try {
    // Try to create lock file (atomic operation)
    const lockData = {
      pid: process.pid,
      startTime: new Date().toISOString()
    };

    // Check if lock file exists and is stale
    try {
      const existing = JSON.parse(await fs.readFile(lockFilePath, 'utf-8'));
      const ageMs = Date.now() - new Date(existing.startTime).getTime();

      if (ageMs < 2 * 3600 * 1000) { // 2 hours
        console.log(`Job already running (PID: ${existing.pid})`);
        return;
      }

      console.log('Stale lock detected, cleaning up');
    } catch (err) {
      // Lock file doesn't exist, which is good
    }

    // Create lock file with PID
    await fs.writeFile(lockFilePath, JSON.stringify(lockData, null, 2));
    console.log('Lock acquired');

    try {
      // Do the work
      await generateDailyReport();
      console.log('Job completed successfully');
    } finally {
      // Clean up lock file
      await fs.unlink(lockFilePath);
      console.log('Lock released');
    }
  } catch (err) {
    console.error('Cron error:', err);
  }
});
```

**Pros:**
- No external dependencies
- Very simple
- Single file

**Cons:**
- Only works on single server
- Not atomic on some filesystems
- Requires cleanup

---

## Strategy 4: Process Lock Check (Safest Single-Server)

Check if another process is running before starting.

```javascript
const cron = require('node-cron');
const { exec } = require('child_process');
const { promisify } = require('util');
const execAsync = promisify(exec);

cron.schedule('0 0 * * *', async () => {
  try {
    // Check if job process is already running
    const { stdout } = await execAsync(
      `pgrep -f "node.*daily-report.js" | wc -l`
    );

    const processCount = parseInt(stdout.trim());

    if (processCount > 1) { // More than this process
      console.log('Job already running');
      return;
    }

    console.log('Starting job');

    // Do the work
    await generateDailyReport();

    console.log('Job completed successfully');
  } catch (err) {
    console.error('Cron error:', err);
  }
});
```

---

## Strategy 5: Monitoring with Database State

Track execution status in database (recommended).

```javascript
const cron = require('node-cron');
const JobExecution = require('./models/JobExecution');

cron.schedule('0 0 * * *', async () => {
  const jobName = 'daily-report';

  try {
    // Check if already running
    const existing = await JobExecution.findOne({
      jobName,
      status: 'running'
    });

    if (existing) {
      const runningFor = Date.now() - existing.startedAt.getTime();
      const maxDuration = 2 * 3600 * 1000; // 2 hours

      if (runningFor > maxDuration) {
        // Assume stuck, mark as failed and restart
        await JobExecution.findByIdAndUpdate(existing._id, {
          status: 'failed',
          error: 'Timeout after ' + maxDuration + 'ms'
        });
      } else {
        // Still running
        console.log('Job already running');
        return;
      }
    }

    // Start execution record
    const execution = await JobExecution.create({
      jobName,
      status: 'running',
      startedAt: new Date()
    });

    try {
      // Do the work
      const result = await generateDailyReport();

      // Mark as completed
      await JobExecution.findByIdAndUpdate(execution._id, {
        status: 'completed',
        completedAt: new Date(),
        result
      });

      console.log('Job completed successfully');
    } catch (err) {
      // Mark as failed
      await JobExecution.findByIdAndUpdate(execution._id, {
        status: 'failed',
        completedAt: new Date(),
        error: err.message
      });

      throw err;
    }
  } catch (err) {
    console.error('Cron error:', err);
  }
});
```

**Pros:**
- Full audit trail
- Can detect stuck jobs
- Can track history
- Supports alerting

**Cons:**
- More complex
- Requires database schema

---

## Best Practice: Combination Approach

```javascript
// Use Redis lock + database tracking
cron.schedule('0 0 * * *', async () => {
  const lockKey = 'cron:daily-report';
  const lockValue = crypto.randomUUID();
  const TTL = 3 * 3600; // 3 hours

  try {
    // 1. Try Redis lock (fast check)
    const lockAcquired = await redisClient.set(
      lockKey,
      lockValue,
      'NX',
      'EX',
      TTL
    );

    if (!lockAcquired) {
      console.log('Redis lock held, skipping');
      return;
    }

    // 2. Create database execution record
    const execution = await JobExecution.create({
      jobName: 'daily-report',
      status: 'running',
      startedAt: new Date()
    });

    try {
      // 3. Do the work
      console.log('Starting daily report generation');
      const result = await generateDailyReport();

      // 4. Mark as completed
      execution.status = 'completed';
      execution.completedAt = new Date();
      execution.result = result;
      await execution.save();

      console.log('Daily report completed successfully');
    } catch (err) {
      // 5. Mark as failed
      execution.status = 'failed';
      execution.completedAt = new Date();
      execution.error = err.message;
      await execution.save();

      throw err;
    }
  } finally {
    // 6. Release Redis lock
    const currentLock = await redisClient.get(lockKey);
    if (currentLock === lockValue) {
      await redisClient.del(lockKey);
    }
  }
});
```

This combines:
- Fast Redis lock for quick check
- Database record for audit trail
- Proper error handling and cleanup
