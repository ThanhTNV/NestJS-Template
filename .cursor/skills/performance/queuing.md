# Job Queuing Patterns

This document covers advanced queuing patterns, job handling, and queue management.

## Queue Types

### 1. FIFO Queues (First In, First Out)

Standard order processing. Use for sequential operations.

```typescript
@Injectable()
export class OrderService {
  constructor(@InjectQueue('orders') private orderQueue: Queue) {}

  async processOrder(orderId: string): Promise<void> {
    // Jobs processed in order
    await this.orderQueue.add(
      'process',
      { orderId },
      { priority: 0 } // Standard priority
    );
  }
}

@Processor('orders')
export class OrderProcessor {
  @Process('process')
  async handle(job: Job) {
    // Processes in order received
    await this.validateOrder(job.data.orderId);
    await this.processPayment(job.data.orderId);
    await this.shipOrder(job.data.orderId);
  }
}
```

**Use when**: Sequential workflows, order processing, batch operations.

### 2. Priority Queues

High-priority jobs jump the line.

```typescript
@Injectable()
export class NotificationService {
  constructor(@InjectQueue('notifications') private queue: Queue) {}

  async sendUrgentNotification(userId: string): Promise<void> {
    await this.queue.add(
      'send',
      { userId, type: 'urgent' },
      { priority: 10 }, // High priority
    );
  }

  async sendRegularNotification(userId: string): Promise<void> {
    await this.queue.add(
      'send',
      { userId, type: 'regular' },
      { priority: 5 }, // Medium priority
    );
  }
}

@Processor('notifications')
export class NotificationProcessor {
  @Process({ name: 'send', concurrency: 5 })
  async handle(job: Job) {
    // Urgent jobs processed first
    await this.notificationService.send(job.data.userId);
  }
}
```

**Use when**: Mixed urgency jobs (alerts, batch reporting).

### 3. Delayed Jobs

Execute after delay.

```typescript
// Send reminder 1 hour after user registration
await this.reminderQueue.add(
  'send-reminder',
  { userId },
  { delay: 3600000 }, // 1 hour
);

// Send daily digest every day at 9 AM
await this.digestQueue.add(
  'send-daily',
  { userId },
  { 
    delay: calculateDelayUntilNine(), 
    repeat: { pattern: '0 9 * * *' } // Cron expression
  },
);
```

**Use when**: Scheduled operations, reminders, reports.

### 4. Repeating Jobs

Recurring jobs on schedule.

```typescript
@Injectable()
export class ReportService {
  constructor(@InjectQueue('reports') private reportQueue: Queue) {}

  async scheduleHourlyReport(): Promise<void> {
    await this.reportQueue.add(
      'generate',
      {},
      {
        repeat: {
          pattern: '0 * * * *', // Every hour
        },
      },
    );
  }

  async scheduleDailyReport(): Promise<void> {
    await this.reportQueue.add(
      'generate-daily',
      {},
      {
        repeat: {
          cron: '0 9 * * *', // 9 AM daily
          tz: 'America/New_York',
        },
      },
    );
  }
}
```

**Use when**: Periodic tasks, cron jobs, cleanup operations.

## Job Failure Handling

### Retry Strategy

```typescript
await this.externalApiQueue.add(
  'call-api',
  { endpoint: '/data' },
  {
    attempts: 5, // Retry 5 times
    backoff: {
      type: 'exponential',
      delay: 2000, // Start at 2 seconds
    }, // Delays: 2s, 4s, 8s, 16s, 32s
  },
);

// Linear backoff
{
  backoff: {
    type: 'fixed',
    delay: 5000, // Always 5 seconds
  },
}
```

### Dead Letter Queue

```typescript
@Processor('emails')
export class EmailProcessor {
  constructor(
    @InjectQueue('emails') private emailQueue: Queue,
    @InjectQueue('dlq') private dlqQueue: Queue, // Dead Letter Queue
    private logger: Logger,
  ) {}

  @Process({ name: 'send', concurrency: 10 })
  async handle(job: Job) {
    try {
      await this.emailService.send(job.data);
    } catch (error) {
      this.logger.error(`Job ${job.id} failed after retries`, error);

      // Move to DLQ for investigation
      await this.dlqQueue.add(
        'process-failure',
        {
          originalJob: job.data,
          error: error.message,
          failedAt: new Date(),
        },
      );

      throw error;
    }
  }
}
```

### Selective Retry

```typescript
@Process('external-api')
async callApi(job: Job) {
  try {
    const result = await this.httpService.get(job.data.url).toPromise();
    return result.data;
  } catch (error) {
    // Retry on temporary failures
    if (error.response?.status === 429 || error.code === 'ECONNREFUSED') {
      throw error; // Triggers retry
    }

    // Don't retry on permanent failures
    if (error.response?.status === 400) {
      this.logger.error('Bad request, not retrying', error);
      return null; // Mark as success (skip)
    }

    throw error;
  }
}
```

## Bulk Operations

### Adding Multiple Jobs

```typescript
@Injectable()
export class BulkEmailService {
  constructor(@InjectQueue('emails') private emailQueue: Queue) {}

  async sendBulkWelcomeEmails(userIds: string[]): Promise<void> {
    const jobs = userIds.map(userId => ({
      name: 'send-welcome',
      data: { userId },
      opts: { 
        attempts: 3,
        backoff: { type: 'exponential', delay: 2000 },
      },
    }));

    // Add all at once
    await this.emailQueue.addBulk(jobs);

    this.logger.log(`Queued ${userIds.length} welcome emails`);
  }
}
```

### Progress Tracking

```typescript
@Injectable()
export class ProgressTrackingService {
  constructor(@InjectQueue('reports') private reportQueue: Queue) {}

  async generateReport(reportId: string): Promise<void> {
    const job = await this.reportQueue.add(
      'generate',
      { reportId },
    );

    // Track progress
    job.progress(0);

    // Store job ID for frontend polling
    await this.redis.set(`report:${reportId}:job`, job.id, 'EX', 3600);
  }
}

@Processor('reports')
export class ReportProcessor {
  @Process('generate')
  async generate(job: Job) {
    const { reportId } = job.data;
    const items = await this.fetchItems();

    for (let i = 0; i < items.length; i++) {
      await this.processItem(items[i]);

      // Update progress
      job.progress((i + 1) / items.length * 100);
    }

    return { reportId, itemsProcessed: items.length };
  }
}

// Frontend polling
@Get('reports/:id/progress')
async getProgress(@Param('id') reportId: string) {
  const jobId = await this.redis.get(`report:${reportId}:job`);
  if (!jobId) return { status: 'not-started' };

  const job = await this.reportQueue.getJob(jobId);
  return {
    status: job.getState(),
    progress: job.progress(),
  };
}
```

## Queue Management

### Monitoring

```typescript
@Injectable()
export class QueueMonitorService {
  constructor(
    @InjectQueue('emails') private emailQueue: Queue,
    @InjectQueue('reports') private reportQueue: Queue,
    private logger: Logger,
  ) {}

  @Scheduled(CronExpression.EVERY_5_MINUTES)
  async checkQueueHealth(): Promise<void> {
    const emailQueueMetrics = await this.getQueueMetrics(this.emailQueue);
    const reportQueueMetrics = await this.getQueueMetrics(this.reportQueue);

    this.logger.log('Queue Metrics', {
      emails: emailQueueMetrics,
      reports: reportQueueMetrics,
    });

    // Alert if queue is backed up
    if (emailQueueMetrics.waiting > 1000) {
      this.alertService.notify('Email queue backed up');
    }
  }

  private async getQueueMetrics(queue: Queue) {
    const [waiting, active, delayed, failed, completed] = await Promise.all([
      queue.getWaitingCount(),
      queue.getActiveCount(),
      queue.getDelayedCount(),
      queue.getFailedCount(),
      queue.getCompletedCount(),
    ]);

    return {
      waiting,
      active,
      delayed,
      failed,
      completed,
      backlog: waiting + delayed,
    };
  }
}
```

### Queue Cleanup

```typescript
@Scheduled(CronExpression.EVERY_DAY_AT_MIDNIGHT)
async cleanupQueues(): Promise<void> {
  const queues = [this.emailQueue, this.reportQueue];

  for (const queue of queues) {
    // Remove completed jobs older than 7 days
    await queue.clean(7 * 24 * 3600 * 1000, 'completed');

    // Remove failed jobs older than 30 days
    await queue.clean(30 * 24 * 3600 * 1000, 'failed');

    this.logger.log(`Cleaned up ${queue.name}`);
  }
}
```

### Pausing/Resuming

```typescript
// Pause during maintenance
await this.emailQueue.pause();
this.logger.log('Email queue paused');

// Resume after maintenance
await this.emailQueue.resume();
this.logger.log('Email queue resumed');
```

## Queue Patterns

### Fan-Out Pattern

One job spawns multiple child jobs.

```typescript
@Processor('tasks')
export class TaskProcessor {
  constructor(
    @InjectQueue('tasks') private taskQueue: Queue,
    @InjectQueue('subtasks') private subtaskQueue: Queue,
  ) {}

  @Process('split-work')
  async splitWork(job: Job) {
    const data = job.data;
    const chunks = this.splitIntoChunks(data, 10);

    // Create sub-jobs
    const subJobs = chunks.map((chunk, index) => ({
      name: 'process-chunk',
      data: { chunk, parentJobId: job.id },
      opts: { jobId: `${job.id}-chunk-${index}` },
    }));

    await this.subtaskQueue.addBulk(subJobs);
  }
}
```

### Fan-In Pattern

Multiple jobs feed into one aggregation job.

```typescript
@Processor('results')
export class ResultAggregator {
  constructor(
    @InjectQueue('results') private resultQueue: Queue,
  ) {}

  @Process('aggregate')
  async aggregate(job: Job) {
    const { jobIds } = job.data;

    // Wait for all child jobs
    const results = await Promise.all(
      jobIds.map(jobId => this.waitForJob(jobId))
    );

    return {
      aggregatedAt: new Date(),
      itemsProcessed: results.length,
      totalTime: results.reduce((sum, r) => sum + r.time, 0),
    };
  }
}
```

## Best Practices

### 1. Idempotent Jobs

Job can be safely retried without side effects.

```typescript
@Process('update-user')
async updateUser(job: Job) {
  const { userId, updates } = job.data;

  // Use idempotency key to prevent duplicates
  const idempotencyKey = `update:${userId}:${JSON.stringify(updates)}`;
  const cached = await this.redis.get(idempotencyKey);
  
  if (cached) {
    return cached; // Already processed
  }

  const result = await this.userService.update(userId, updates);
  await this.redis.set(idempotencyKey, result, 'EX', 3600);

  return result;
}
```

### 2. Job Serialization

Keep job data small and serializable.

```typescript
// ❌ BAD - Large nested object
await this.queue.add('process', largeUserObject);

// ✅ GOOD - Just the ID
await this.queue.add('process', { userId: user.id });
```

### 3. Timeout Configuration

Prevent jobs from running indefinitely.

```typescript
@Processor('reports')
export class ReportProcessor {
  @Process({ name: 'generate', timeout: 300000 }) // 5 minutes
  async generate(job: Job) {
    // Must complete within 5 minutes
    return await this.generateReport(job.data);
  }
}
```

### 4. Concurrency Control

Balance throughput vs. resource usage.

```typescript
@Processor('emails')
export class EmailProcessor {
  @Process({ name: 'send', concurrency: 5 }) // Max 5 concurrent
  async send(job: Job) {
    // Only 5 jobs run simultaneously
    return await this.emailService.send(job.data);
  }
}
```
