# Complete Implementation: Redis, Database & File Locks

## Production-Ready Redis Lock

```javascript
const redis = require('redis');
const { promisify } = require('util');

class CronJobLock {
  constructor(redisClient, lockKey, durationSeconds) {
    this.client = redisClient;
    this.lockKey = lockKey;
    this.durationSeconds = durationSeconds;
    this.lockValue = null;
    this.renewInterval = null;
  }

  async acquire() {
    // Generate unique lock value
    this.lockValue = `${Date.now()}-${Math.random()}`;

    const result = await this.client.set(
      this.lockKey,
      this.lockValue,
      'NX',
      'EX',
      this.durationSeconds
    );

    if (!result) {
      return false;
    }

    // Renew lock every 30 seconds
    this.renewInterval = setInterval(async () => {
      try {
        await this.renew();
      } catch (err) {
        console.error('Failed to renew lock:', err);
      }
    }, 30000);

    return true;
  }

  async renew() {
    // Check if still owning the lock before renewing
    const current = await this.client.get(this.lockKey);
    if (current !== this.lockValue) {
      throw new Error('Lock not owned');
    }

    await this.client.set(
      this.lockKey,
      this.lockValue,
      'EX',
      this.durationSeconds
    );
  }

  async release() {
    if (this.renewInterval) {
      clearInterval(this.renewInterval);
    }

    // Only release if we still own it
    const current = await this.client.get(this.lockKey);
    if (current === this.lockValue) {
      await this.client.del(this.lockKey);
    }
  }
}

// Usage with cron
const cron = require('node-cron');
const redis = require('redis');
const client = redis.createClient();

cron.schedule('0 0 * * *', async () => {
  const lock = new CronJobLock(client, 'cron:daily-report', 7200); // 2 hours

  try {
    if (!await lock.acquire()) {
      console.log('Could not acquire lock, job already running');
      return;
    }

    console.log('Lock acquired, starting daily report');

    // Do work
    await dailyReportJob();

    console.log('Daily report completed');
  } catch (err) {
    console.error('Job failed:', err);
  } finally {
    await lock.release();
  }
});

async function dailyReportJob() {
  // Long-running job
  console.log('Processing transactions...');
  await new Promise(resolve => setTimeout(resolve, 5000));
  console.log('Generating report...');
  await new Promise(resolve => setTimeout(resolve, 5000));
}
```

## PostgreSQL Advisory Lock Implementation

```javascript
const { Pool } = require('pg');

class PostgresLock {
  constructor(poolConfig, lockId = 12345) {
    this.pool = new Pool(poolConfig);
    this.lockId = lockId;
    this.acquired = false;
  }

  async acquire(timeoutMs = 5000) {
    const client = await this.pool.connect();

    try {
      // Set timeout for lock acquisition
      await client.query(`SET lock_timeout = '${timeoutMs}ms'`);

      // Try advisory lock
      const result = await client.query(
        'SELECT pg_try_advisory_lock($1) as acquired',
        [this.lockId]
      );

      if (result.rows[0].acquired) {
        this.client = client;
        this.acquired = true;
        console.log(`Lock ${this.lockId} acquired`);
        return true;
      } else {
        client.release();
        console.log(`Lock ${this.lockId} already held`);
        return false;
      }
    } catch (err) {
      client.release();
      throw err;
    }
  }

  async release() {
    if (!this.acquired || !this.client) return;

    try {
      await this.client.query(
        'SELECT pg_advisory_unlock($1)',
        [this.lockId]
      );
      console.log(`Lock ${this.lockId} released`);
    } finally {
      this.client.release();
      this.acquired = false;
    }
  }

  async executeWithinLock(fn) {
    try {
      if (!await this.acquire()) {
        return null;
      }

      return await fn(this.client);
    } finally {
      await this.release();
    }
  }
}

// Usage
const cron = require('node-cron');

cron.schedule('0 0 * * *', async () => {
  const lock = new PostgresLock({
    user: 'postgres',
    password: process.env.DB_PASSWORD,
    host: 'localhost',
    database: 'myapp'
  }, 12345);

  const result = await lock.executeWithinLock(async (client) => {
    console.log('Starting daily report within lock');

    // Run queries with the same client
    const yesterday = new Date();
    yesterday.setDate(yesterday.getDate() - 1);

    const transactions = await client.query(
      'SELECT * FROM transactions WHERE DATE(created_at) = $1',
      [yesterday]
    );

    const summary = {
      date: yesterday,
      count: transactions.rows.length,
      total: transactions.rows.reduce((sum, t) => sum + t.amount, 0)
    };

    await client.query(
      `INSERT INTO daily_reports (report_date, transaction_count, total_amount)
       VALUES ($1, $2, $3)`,
      [summary.date, summary.count, summary.total]
    );

    console.log('Daily report completed');
    return summary;
  });

  if (result) {
    console.log('Report created:', result);
  } else {
    console.log('Job already running');
  }
});
```

## Database Execution Tracking

```javascript
const mongoose = require('mongoose');

const jobExecutionSchema = new mongoose.Schema({
  jobName: { type: String, required: true, index: true },
  status: {
    type: String,
    enum: ['pending', 'running', 'completed', 'failed'],
    default: 'running'
  },
  startedAt: { type: Date, required: true, default: Date.now },
  completedAt: { type: Date },
  duration: { type: Number }, // ms
  result: mongoose.Schema.Types.Mixed,
  error: { type: String },
  logs: [String]
}, { timestamps: true });

// Ensure only one job can be running at a time
jobExecutionSchema.index(
  { jobName: 1, status: 1 },
  { sparse: true, unique: true, partialFilterExpression: { status: 'running' } }
);

const JobExecution = mongoose.model('JobExecution', jobExecutionSchema);

class CronJobExecutor {
  constructor(jobName, maxConcurrent = 1) {
    this.jobName = jobName;
    this.maxConcurrent = maxConcurrent;
  }

  async executeWithTracking(jobFn) {
    let execution = null;

    try {
      // Check for existing running executions
      const running = await JobExecution.countDocuments({
        jobName: this.jobName,
        status: 'running'
      });

      if (running >= this.maxConcurrent) {
        console.log(`${this.jobName} already running (${running}/${this.maxConcurrent})`);
        return null;
      }

      // Create execution record
      execution = await JobExecution.create({
        jobName: this.jobName,
        status: 'running',
        logs: []
      });

      console.log(`Starting ${this.jobName} (execution ${execution._id})`);

      const startTime = Date.now();

      // Execute job with logging
      const result = await jobFn({
        log: (msg) => {
          console.log(`[${this.jobName}] ${msg}`);
          execution.logs.push(msg);
        }
      });

      const duration = Date.now() - startTime;

      // Mark as completed
      execution.status = 'completed';
      execution.completedAt = new Date();
      execution.duration = duration;
      execution.result = result;
      await execution.save();

      console.log(`${this.jobName} completed in ${duration}ms`);

      return result;
    } catch (err) {
      console.error(`${this.jobName} failed:`, err);

      if (execution) {
        execution.status = 'failed';
        execution.completedAt = new Date();
        execution.error = err.message;
        execution.logs.push(`ERROR: ${err.message}\n${err.stack}`);
        await execution.save();
      }

      throw err;
    }
  }

  async getHistory(limit = 10) {
    return JobExecution.find({ jobName: this.jobName })
      .sort({ startedAt: -1 })
      .limit(limit);
  }

  async getStats() {
    const executions = await JobExecution.find({ jobName: this.jobName });

    const stats = {
      total: executions.length,
      completed: 0,
      failed: 0,
      averageDuration: 0,
      lastRun: null
    };

    let totalDuration = 0;

    executions.forEach(exec => {
      if (exec.status === 'completed') stats.completed++;
      if (exec.status === 'failed') stats.failed++;
      if (exec.duration) totalDuration += exec.duration;
      if (!stats.lastRun || exec.startedAt > stats.lastRun.startedAt) {
        stats.lastRun = exec;
      }
    });

    stats.averageDuration = Math.round(totalDuration / stats.completed);
    stats.successRate = ((stats.completed / stats.total) * 100).toFixed(2) + '%';

    return stats;
  }
}

// Usage with cron
const cron = require('node-cron');
const executor = new CronJobExecutor('daily-report');

cron.schedule('0 0 * * *', async () => {
  try {
    const result = await executor.executeWithTracking(async (context) => {
      context.log('Fetching transactions...');
      const count = await countTransactions();

      context.log(`Processing ${count} transactions`);
      await processTransactions();

      context.log('Generating report...');
      const report = await generateReport();

      return report;
    });

    if (result) {
      // Send alert, update dashboard, etc.
      console.log('Execution result:', result);
    }
  } catch (err) {
    console.error('Execution failed:', err);
    // Send alert to monitoring system
  }
});

// Monitoring endpoint
app.get('/api/cron-status', async (req, res) => {
  const executor = new CronJobExecutor('daily-report');
  const history = await executor.getHistory(5);
  const stats = await executor.getStats();

  res.json({
    stats,
    recentExecutions: history
  });
});
```

## Complete Example: All Strategies Combined

```javascript
const cron = require('node-cron');
const redis = require('redis');
const { Pool } = require('pg');
const mongoose = require('mongoose');

class RobustCronJob {
  constructor(jobName, options = {}) {
    this.jobName = jobName;
    this.redisClient = redis.createClient(options.redis);
    this.pgPool = new Pool(options.postgres);
    this.lockKey = `cron:${jobName}`;
    this.lockValue = null;
  }

  async run(jobFn) {
    const startTime = Date.now();

    try {
      // 1. Try Redis lock (fastest)
      if (!await this.acquireRedisLock()) {
        console.log(`${this.jobName}: Already running (Redis lock)`);
        return;
      }

      // 2. Create database record
      const execution = await this.createExecutionRecord('running');

      try {
        // 3. Run the job
        console.log(`${this.jobName}: Started`);
        const result = await jobFn();

        // 4. Mark as successful
        await this.updateExecutionRecord(execution._id, {
          status: 'completed',
          result,
          duration: Date.now() - startTime
        });

        console.log(`${this.jobName}: Completed in ${Date.now() - startTime}ms`);
        return result;
      } catch (err) {
        // Mark as failed
        await this.updateExecutionRecord(execution._id, {
          status: 'failed',
          error: err.message,
          duration: Date.now() - startTime
        });

        throw err;
      }
    } finally {
      // 5. Release Redis lock
      await this.releaseRedisLock();
    }
  }

  async acquireRedisLock() {
    this.lockValue = Date.now().toString();
    const acquired = await this.redisClient.set(
      this.lockKey,
      this.lockValue,
      'NX',
      'EX',
      7200 // 2 hours
    );
    return acquired;
  }

  async releaseRedisLock() {
    if (!this.lockValue) return;
    const current = await this.redisClient.get(this.lockKey);
    if (current === this.lockValue) {
      await this.redisClient.del(this.lockKey);
    }
  }

  async createExecutionRecord(status) {
    const JobExecution = mongoose.model('JobExecution');
    return JobExecution.create({
      jobName: this.jobName,
      status,
      startedAt: new Date()
    });
  }

  async updateExecutionRecord(id, updates) {
    const JobExecution = mongoose.model('JobExecution');
    return JobExecution.findByIdAndUpdate(
      id,
      { ...updates, completedAt: new Date() },
      { new: true }
    );
  }
}

// Usage
const robust = new RobustCronJob('daily-report', {
  redis: { host: 'localhost' },
  postgres: { /* config */ }
});

cron.schedule('0 0 * * *', async () => {
  try {
    await robust.run(async () => {
      // Your job logic here
      console.log('Processing...');
      await new Promise(r => setTimeout(r, 5000));
      return { processed: 1000 };
    });
  } catch (err) {
    console.error('Job failed:', err);
    // Send alert
  }
});
```
