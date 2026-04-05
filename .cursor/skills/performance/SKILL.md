---
name: performance
description: Optimize NestJS application performance using Redis caching, Bull job queues, and database query optimization. Prevent N+1 queries, add proper indexing, and implement caching strategies. Use when implementing features, optimizing queries, or scaling workloads.
---

# Performance & Scalability

This skill guides implementing performance optimizations in NestJS applications: caching strategies, asynchronous job queues, and database query optimization.

## Caching Strategies

### Setup Redis Caching

```bash
npm install @nestjs/cache-manager cache-manager redis
```

### Module Configuration

```typescript
// src/cache/cache.module.ts
import { Module } from '@nestjs/common';
import { CacheModule } from '@nestjs/cache-manager';
import * as redisStore from 'cache-manager-redis-store';

@Module({
  imports: [
    CacheModule.register({
      isGlobal: true,
      store: redisStore,
      host: process.env.REDIS_HOST || 'localhost',
      port: process.env.REDIS_PORT || 6379,
      ttl: 60 * 5, // 5 minutes default
    }),
  ],
})
export class CacheConfigModule {}
```

### Service-Level Caching

```typescript
// src/users/user.service.ts
import { Injectable, Inject } from '@nestjs/common';
import { CACHE_MANAGER } from '@nestjs/cache-manager';
import { Cache } from 'cache-manager';

@Injectable()
export class UserService {
  constructor(
    private readonly userRepository: UserRepository,
    @Inject(CACHE_MANAGER) private cacheManager: Cache,
    private readonly logger: Logger,
  ) {}

  async getUserById(id: string): Promise<User> {
    // Check cache first
    const cached = await this.cacheManager.get<User>(`user:${id}`);
    if (cached) {
      this.logger.debug(`Cache hit: user:${id}`);
      return cached;
    }

    // Query database
    const user = await this.userRepository.findById(id);
    if (!user) {
      throw new UserNotFoundError(id);
    }

    // Store in cache
    await this.cacheManager.set(`user:${id}`, user, 60 * 5); // 5 minutes

    return user;
  }

  async updateUser(id: string, updates: Partial<User>): Promise<User> {
    const user = await this.userRepository.update(id, updates);

    // Invalidate cache
    await this.cacheManager.del(`user:${id}`);

    this.logger.debug(`Cache invalidated: user:${id}`);

    return user;
  }

  async deleteUser(id: string): Promise<void> {
    await this.userRepository.delete(id);
    
    // Invalidate cache
    await this.cacheManager.del(`user:${id}`);
  }
}
```

### Cache Decorator

```typescript
// src/common/decorators/cache.decorator.ts
import { Cacheable } from '@nestjs/cache-manager';
import { Inject } from '@nestjs/common';
import { CACHE_MANAGER } from '@nestjs/cache-manager';
import { Cache } from 'cache-manager';

export function CacheResult(ttl: number = 300) {
  return function (
    target: any,
    propertyKey: string,
    descriptor: PropertyDescriptor,
  ) {
    const originalMethod = descriptor.value;

    descriptor.value = async function (...args: any[]) {
      const cache = this.cacheManager as Cache;
      const key = `${target.constructor.name}:${propertyKey}:${JSON.stringify(args)}`;

      const cached = await cache.get(key);
      if (cached) return cached;

      const result = await originalMethod.apply(this, args);
      await cache.set(key, result, ttl);

      return result;
    };

    return descriptor;
  };
}

// Usage
@Injectable()
export class ProductService {
  @CacheResult(600) // 10 minutes
  async getProductsByCategory(categoryId: string): Promise<Product[]> {
    return this.productRepository.findByCategory(categoryId);
  }
}
```

## Job Queues with Bull

### Setup Bull Queues

```bash
npm install @nestjs/bull bull @nestjs/ioredis redis
```

### Queue Configuration

```typescript
// src/queue/queue.module.ts
import { Module } from '@nestjs/common';
import { BullModule } from '@nestjs/bull';

@Module({
  imports: [
    BullModule.forRoot({
      redis: {
        host: process.env.REDIS_HOST || 'localhost',
        port: parseInt(process.env.REDIS_PORT || '6379'),
      },
    }),
    BullModule.registerQueue(
      { name: 'emails' },
      { name: 'reports' },
      { name: 'analytics' },
    ),
  ],
})
export class QueueModule {}
```

### Producer: Queue Jobs

```typescript
// src/email/email.service.ts
import { Injectable } from '@nestjs/common';
import { Queue } from 'bull';
import { InjectQueue } from '@nestjs/bull';

@Injectable()
export class EmailService {
  constructor(@InjectQueue('emails') private emailQueue: Queue) {}

  async sendWelcomeEmail(userId: string, email: string): Promise<void> {
    // Add to queue instead of sending immediately
    await this.emailQueue.add(
      'send-welcome',
      { userId, email },
      {
        delay: 0,
        attempts: 3,
        backoff: {
          type: 'exponential',
          delay: 2000,
        },
      },
    );

    this.logger.log(`Email queued for ${email}`);
  }

  async sendBulkEmails(userIds: string[]): Promise<void> {
    const jobs = userIds.map(id => ({
      data: { userId: id },
      opts: { attempts: 3 },
    }));

    await this.emailQueue.addBulk(jobs);

    this.logger.log(`${userIds.length} emails queued`);
  }
}
```

### Consumer: Process Jobs

```typescript
// src/email/email.processor.ts
import { Processor, Process } from '@nestjs/bull';
import { Job } from 'bull';

@Processor('emails')
export class EmailProcessor {
  constructor(private readonly emailProvider: EmailProvider) {}

  @Process('send-welcome')
  async handleSendWelcome(job: Job) {
    const { userId, email } = job.data;

    try {
      // Send email
      await this.emailProvider.sendWelcome(email);

      this.logger.log(`Welcome email sent: ${email}`);

      return { success: true, email };
    } catch (error) {
      this.logger.error(`Failed to send email: ${email}`, error);
      throw error; // Triggers retry
    }
  }

  @Process('send-reminder')
  async handleSendReminder(job: Job) {
    const { email, subject, body } = job.data;

    await this.emailProvider.send(email, subject, body);

    return { success: true };
  }
}
```

## Database Query Optimization

### N+1 Query Prevention

```typescript
// ❌ BAD - N+1 queries
async getAllOrders(): Promise<Order[]> {
  const orders = await this.orderRepository.find();
  
  // Executes N queries (one per order)
  const enriched = await Promise.all(
    orders.map(order =>
      this.orderRepository.findWithItems(order.id)
    )
  );

  return enriched;
}

// ✅ GOOD - Single query with eager loading
async getAllOrders(): Promise<Order[]> {
  return this.orderRepository.find({
    relations: ['items', 'customer', 'payments'],
    select: {
      id: true,
      total: true,
      customer: { id: true, name: true },
      items: { id: true, quantity: true, price: true },
    },
  });
}
```

### Database Indexes

```typescript
// src/users/entities/user.entity.ts
import { Entity, Index } from 'typeorm';

@Entity('users')
@Index('idx_email', ['email'], { unique: true })
@Index('idx_created_at', ['createdAt'])
@Index('idx_status', ['status'])
@Index('idx_email_status', ['email', 'status']) // Composite
export class User {
  @PrimaryGeneratedColumn('uuid')
  id: string;

  @Column({ unique: true })
  email: string;

  @Column()
  name: string;

  @Column()
  status: 'active' | 'inactive';

  @CreateDateColumn()
  createdAt: Date;
}
```

### Query Optimization Patterns

```typescript
// Use pagination for large result sets
async getUsers(page: number = 1, limit: number = 10): Promise<PaginatedResult<User>> {
  const skip = (page - 1) * limit;

  const [users, total] = await this.userRepository.findAndCount({
    skip,
    take: limit,
    order: { createdAt: 'DESC' },
  });

  return {
    data: users,
    total,
    page,
    limit,
    pages: Math.ceil(total / limit),
  };
}

// Use select to fetch only needed fields
async getUserEmails(): Promise<{ id: string; email: string }[]> {
  return this.userRepository.find({
    select: { id: true, email: true }, // Don't fetch large fields
  });
}

// Use QueryBuilder for complex queries
async getActiveUsersByRegion(region: string): Promise<User[]> {
  return this.userRepository
    .createQueryBuilder('user')
    .where('user.status = :status', { status: 'active' })
    .andWhere('user.region = :region', { region })
    .orderBy('user.createdAt', 'DESC')
    .limit(100)
    .getMany();
}
```

## Performance Monitoring

### Query Performance

```typescript
// src/common/interceptors/query-performance.interceptor.ts
import { Injectable, NestInterceptor } from '@nestjs/common';
import { Observable } from 'rxjs';
import { tap } from 'rxjs/operators';

@Injectable()
export class QueryPerformanceInterceptor implements NestInterceptor {
  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    const start = Date.now();

    return next.handle().pipe(
      tap(() => {
        const duration = Date.now() - start;

        if (duration > 1000) {
          // Log slow queries
          this.logger.warn(`Slow query: ${duration}ms`);
        }
      }),
    );
  }
}
```

### Cache Hit Rates

```typescript
// Monitor cache effectiveness
@Injectable()
export class CacheMetricsService {
  private hits = 0;
  private misses = 0;

  recordHit(): void {
    this.hits++;
  }

  recordMiss(): void {
    this.misses++;
  }

  getHitRate(): number {
    const total = this.hits + this.misses;
    return total === 0 ? 0 : (this.hits / total) * 100;
  }

  getMetrics() {
    return {
      hits: this.hits,
      misses: this.misses,
      hitRate: `${this.getHitRate().toFixed(2)}%`,
    };
  }
}
```

## Optimization Checklist

When implementing performance:

- [ ] Identify heavy/frequently accessed operations
- [ ] Implement caching for read-heavy operations
- [ ] Use queues for time-consuming tasks
- [ ] Add database indexes on frequently queried fields
- [ ] Use eager loading to prevent N+1 queries
- [ ] Implement pagination for large result sets
- [ ] Monitor query performance (log slow queries)
- [ ] Use select to fetch only needed columns
- [ ] Implement connection pooling
- [ ] Monitor cache hit rates
- [ ] Set appropriate TTLs for cache entries
- [ ] Test at scale before production

## Review Checklist

When reviewing performance code:

- [ ] Heavy operations moved to queues
- [ ] Cache keys are predictable and isolated
- [ ] Cache is invalidated on data changes
- [ ] Database queries have proper indexes
- [ ] No N+1 queries (eager loading used)
- [ ] Pagination implemented for large result sets
- [ ] Query performance acceptable (<100ms)
- [ ] Cache hit rate monitored (target >70%)
- [ ] Queue jobs have retry logic
- [ ] Failed jobs logged and alertable
- [ ] Slow query warnings configured
- [ ] Timeouts set appropriately

## Additional Resources

- For caching strategies and patterns, see [caching.md](caching.md)
- For queue patterns and job handling, see [queuing.md](queuing.md)
- For database optimization details, see [database.md](database.md)
