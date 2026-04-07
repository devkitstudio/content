# Queue-Based Async Architecture

## Core Pattern: Decouple with Message Queue

```
API Request → Queue → Workers → Completion
├─ Fast: API returns immediately
├─ Reliable: Queue persists jobs
├─ Scalable: Spin up workers as needed
└─ Observable: Track job status
```

## 1. Enqueue Jobs in API Handler

```javascript
const bullQueue = require('bull');
const emailQueue = new bullQueue('emails', {
  redis: {
    host: process.env.REDIS_HOST,
    port: 6379
  }
});

// API endpoint - returns immediately
app.post('/api/campaigns/:id/send', async (req, res) => {
  const campaign = await Campaign.findById(req.params.id);
  const recipients = await getRecipients(campaign.id);

  // Create job for each email
  const jobs = recipients.map(recipient => ({
    campaignId: campaign.id,
    recipientId: recipient.id,
    email: recipient.email,
    subject: campaign.subject,
    template: campaign.template,
    variables: campaign.variables
  }));

  // Add to queue - returns immediately
  await emailQueue.addBulk(
    jobs.map(job => ({
      data: job,
      attempts: 3,
      backoff: {
        type: 'exponential',
        delay: 2000
      },
      removeOnComplete: true,
      timeout: 30000
    }))
  );

  res.json({
    status: 'queued',
    totalJobs: jobs.length,
    campaignId: campaign.id
  });
});
```

## 2. Process Jobs with Workers

```javascript
const nodemailer = require('nodemailer');
const mailer = nodemailer.createTransport({
  service: 'gmail',
  auth: {
    user: process.env.EMAIL_USER,
    pass: process.env.EMAIL_PASS
  }
});

// Worker process (separate service or thread)
emailQueue.process(5, async (job) => {
  const { recipientId, email, subject, template, variables } = job.data;

  try {
    // Render email from template
    const html = await renderTemplate(template, variables);

    // Send email
    await mailer.sendMail({
      from: process.env.FROM_EMAIL,
      to: email,
      subject,
      html
    });

    // Record success
    await EmailLog.create({
      recipientId,
      campaignId: job.data.campaignId,
      status: 'sent',
      sentAt: new Date()
    });

    return { success: true, email };
  } catch (error) {
    // Log error for retry
    console.error(`Failed to send email to ${email}:`, error);

    // Increment job attempt counter - will retry up to 3 times
    throw error; // Re-throw to trigger retry
  }
});

// Handle job completion
emailQueue.on('completed', (job, result) => {
  console.log(`Job ${job.id} completed:`, result);
});

// Handle job failure
emailQueue.on('failed', (job, err) => {
  console.error(`Job ${job.id} failed:`, err.message);
  // Alert admin after 3 retries
  notifyFailedJob(job, err);
});
```

## 3. Monitor Job Progress

```javascript
// Track campaign progress
app.get('/api/campaigns/:id/status', async (req, res) => {
  const campaignId = req.params.id;

  // Get all jobs for this campaign
  const jobs = await emailQueue.getJobs(['active', 'waiting', 'completed', 'failed']);
  const campaignJobs = jobs.filter(j => j.data.campaignId === campaignId);

  const stats = {
    total: campaignJobs.length,
    pending: campaignJobs.filter(j => j.state === 'waiting' || j.state === 'active').length,
    completed: campaignJobs.filter(j => j.state === 'completed').length,
    failed: campaignJobs.filter(j => j.state === 'failed').length
  };

  // Get actual email log
  const logs = await EmailLog.aggregate([
    { $match: { campaignId: campaignId } },
    {
      $group: {
        _id: '$status',
        count: { $sum: 1 }
      }
    }
  ]);

  res.json({
    campaign: campaignId,
    queueStats: stats,
    emailStats: logs
  });
});
```

## 4. Scale with Multiple Workers

```javascript
// worker1.js
const emailQueue = new bullQueue('emails', redisConfig);
emailQueue.process(10, emailJobHandler);

// worker2.js
const emailQueue = new bullQueue('emails', redisConfig);
emailQueue.process(10, emailJobHandler);

// worker3.js
const emailQueue = new bullQueue('emails', redisConfig);
emailQueue.process(10, emailJobHandler);

// All workers pull from same queue
// Bull handles distribution automatically
```

## 5. Handle Dead Letter Queue (DLQ)

```javascript
// Create DLQ for permanently failed jobs
const dlq = new bullQueue('emails-dlq', redisConfig);

emailQueue.on('failed', async (job, err) => {
  // After max retries, move to DLQ
  if (job.attemptsMade >= job.opts.attempts) {
    await dlq.add({
      originalJob: job.data,
      error: err.message,
      failedAt: new Date(),
      attempts: job.attemptsMade
    });

    // Alert admin
    await sendAlert(`Email job failed permanently: ${job.id}`);
  }
});

// Manual retry of DLQ jobs
app.post('/api/admin/retry-dlq', async (req, res) => {
  const dlqJobs = await dlq.getJobs();
  const retried = [];

  for (const dlqJob of dlqJobs) {
    await emailQueue.add(dlqJob.data.originalJob, {
      attempts: 3,
      backoff: { type: 'exponential', delay: 2000 }
    });
    retried.push(dlqJob.id);
  }

  // Clear DLQ
  await dlq.clean(0);

  res.json({ retriedCount: retried.length });
});
```

## 6. Graceful Shutdown

```javascript
// Ensure all jobs complete before shutdown
process.on('SIGTERM', async () => {
  console.log('Shutting down gracefully...');

  // Stop accepting new jobs
  await emailQueue.pause();

  // Wait for active jobs to finish (with timeout)
  const timeout = 30000; // 30 seconds
  const startTime = Date.now();

  while (true) {
    const activeJobs = await emailQueue.getActiveCount();
    if (activeJobs === 0) break;

    if (Date.now() - startTime > timeout) {
      console.warn('Timeout waiting for jobs, forcing shutdown');
      break;
    }

    await new Promise(resolve => setTimeout(resolve, 1000));
  }

  // Close connections
  await emailQueue.close();
  process.exit(0);
});
```
