# Caching Strategies

This document covers advanced caching patterns and strategy selection.

## Caching Strategies

### 1. Write-Through Caching

Data written to cache and database simultaneously. Ensures consistency.

**Use when**: Critical data where consistency is paramount (user accounts, orders).

```typescript
async updateUser(id: string, updates: Partial<User>): Promise<User> {
  // Update database first
  const user = await this.userRepository.update(id, updates);

  // Then update cache
  await this.cacheManager.set(`user:${id}`, user, 300);

  return user;
}
```

**Pros**: Strong consistency, data always in sync
**Cons**: Slower writes (two operations)

### 2. Write-Behind (Write-Back) Caching

Data written to cache first, asynchronously flushed to database.

**Use when**: High-volume writes where immediate consistency isn't critical (activity logs, counters).

```typescript
async recordPageView(userId: string, pageId: string): Promise<void> {
  const key = `view:${userId}:${pageId}`;

  // Write to cache immediately
  const count = await this.cacheManager.get<number>(key) || 0;
  await this.cacheManager.set(key, count + 1, 3600);

  // Queue database write (asynchronous)
  await this.viewQueue.add({
    userId,
    pageId,
    timestamp: new Date(),
  });
}
```

**Pros**: Fast writes, handles bursts well
**Cons**: Data loss risk if crash before flush, eventual consistency

### 3. Refresh-Ahead Caching

Proactively refresh cache before expiration.

**Use when**: Data that's expensive to compute and accessed frequently (dashboard metrics).

```typescript
@Scheduled(CronExpression.EVERY_30_MINUTES)
async refreshDashboardMetrics(): Promise<void> {
  const metrics = await this.calculateMetrics();

  // Refresh before expiration
  await this.cacheManager.set('dashboard:metrics', metrics, 3600);

  this.logger.log('Dashboard metrics refreshed');
}
```

**Pros**: No cache misses for popular data
**Cons**: Wasted computation if data not accessed, complexity

### 4. Time-Based Expiration (TTL)

Cache expires after fixed duration. Most common approach.

```typescript
// 5 minutes TTL
await this.cacheManager.set('user:123', user, 300);
```

**Pros**: Simple, predictable
**Cons**: Stale data possible, requires fine-tuning TTL

### 5. Event-Based Invalidation

Cache invalidated on specific events (data changes).

```typescript
@EventListener()
async onUserUpdated(event: UserUpdatedEvent) {
  await this.cacheManager.del(`user:${event.userId}`);
  await this.cacheManager.del('users:list'); // Invalidate related caches
}
```

**Pros**: Fresh data, no premature eviction
**Cons**: Requires coordination, complex dependencies

## Deciding When to Cache

### Good Caching Candidates

- **High read-to-write ratio**: Product catalog (100:1 reads to writes)
- **Expensive computation**: Complex reports, ML predictions
- **Frequently accessed**: User profiles, settings
- **Static/slow-changing**: Configuration, reference data
- **Network-heavy**: Calls to external APIs

### Poor Caching Candidates

- **High write frequency**: Real-time stock prices (changing constantly)
- **Low access frequency**: Rarely accessed data
- **High consistency requirement**: Financial transactions
- **Small computation cost**: Simple lookups
- **Unique per-user**: Personalized data

## Cache Keys

### Key Design Patterns

```typescript
// User data
`user:${userId}`
`user:${userId}:profile`
`user:${userId}:settings`

// Collection data
`products:category:${categoryId}`
`products:category:${categoryId}:page:${page}`

// Time-series data
`metrics:daily:${date}`
`metrics:hourly:${date}:${hour}`

// Hierarchical
`tenant:${tenantId}:user:${userId}`
`tenant:${tenantId}:user:${userId}:preferences`
```

### Cache Key Namespacing

Prevent collisions and enable bulk operations:

```typescript
class CacheKeyBuilder {
  static user(id: string): string {
    return `user:${id}`;
  }

  static userProfile(id: string): string {
    return `user:${id}:profile`;
  }

  static userPermissions(id: string): string {
    return `user:${id}:permissions`;
  }

  static userPattern(id: string): string {
    return `user:${id}:*`;
  }

  // Bulk invalidation
  static async invalidateUser(
    cache: Cache,
    userId: string,
  ): Promise<void> {
    const keys = await cache.store.getKeys(`${CacheKeyBuilder.userPattern(userId)}`);
    await Promise.all(keys.map(key => cache.del(key)));
  }
}
```

## Distributed Caching

### Multi-Instance Coordination

When running multiple app instances:

```typescript
// Use Redis pub/sub for invalidation across instances
@Injectable()
export class DistributedCacheService {
  constructor(
    @Inject(CACHE_MANAGER) private cache: Cache,
    private redis: Redis,
  ) {
    // Listen for invalidation messages
    this.redis.subscribe('cache:invalidate', (err, count) => {
      if (err) this.logger.error('Failed to subscribe', err);
    });

    this.redis.on('message', (channel, message) => {
      if (channel === 'cache:invalidate') {
        this.cache.del(message); // Invalidate locally
      }
    });
  }

  async invalidateAcrossInstances(key: string): Promise<void> {
    // Publish to all instances
    await this.redis.publish('cache:invalidate', key);
  }
}
```

## Cache Warming

Pre-load cache on startup:

```typescript
@Injectable()
export class CacheWarmingService implements OnModuleInit {
  constructor(
    @Inject(CACHE_MANAGER) private cache: Cache,
    private configService: ConfigService,
  ) {}

  async onModuleInit() {
    this.logger.log('Warming cache...');

    // Pre-load critical data
    const config = await this.configService.getAll();
    await this.cache.set('app:config', config, 3600);

    // Pre-load static data
    const categories = await this.categoryRepository.find();
    await this.cache.set('products:categories', categories, 86400);

    this.logger.log('Cache warmed');
  }
}
```

## Monitoring Cache Health

```typescript
@Injectable()
export class CacheHealthService {
  async checkCacheHealth(): Promise<HealthIndicatorResult> {
    try {
      const cache = this.cacheManager as any;
      
      // Test write
      const testKey = 'health:test';
      await cache.set(testKey, { timestamp: Date.now() }, 60);
      
      // Test read
      const testValue = await cache.get(testKey);
      await cache.del(testKey);

      return this.health.healthy('cache', {
        readable: !!testValue,
        writable: true,
      });
    } catch (error) {
      return this.health.unhealthy('cache', {
        message: error.message,
      });
    }
  }

  async getMetrics(): Promise<CacheMetrics> {
    // Get Redis info for metrics
    const info = await this.redis.info('stats');
    
    return {
      hits: parseInt(info.keyspace_hits),
      misses: parseInt(info.keyspace_misses),
      hitRate: (hits / (hits + misses)) * 100,
      memoryUsed: info.used_memory_human,
    };
  }
}
```

## Anti-Patterns

### 1. Cache Stampede

Multiple requests for expired key, all hit database simultaneously.

```typescript
// ❌ BAD
async getExpensiveData(id: string) {
  const cached = await this.cache.get(`data:${id}`);
  if (!cached) {
    // All concurrent requests compute independently
    const data = await this.expensiveComputation(id);
    await this.cache.set(`data:${id}`, data, 300);
    return data;
  }
  return cached;
}

// ✅ GOOD - Implement probabilistic early expiration
async getExpensiveData(id: string) {
  const cached = await this.cache.get<{ data: any; exp: number }>(`data:${id}`);
  
  if (!cached) {
    const data = await this.expensiveComputation(id);
    const exp = Date.now() + 300 * 1000;
    await this.cache.set(`data:${id}`, { data, exp }, 360);
    return data;
  }

  // Probabilistically refresh before expiration
  if (cached.exp - Date.now() < 60 * 1000 && Math.random() < 0.1) {
    // One thread refreshes asynchronously
    this.refreshAsync(`data:${id}`);
  }

  return cached.data;
}
```

### 2. Distributed Invalidation Lag

Cache inconsistent across instances.

```typescript
// ✅ Use pub/sub for real-time invalidation
async updateUser(userId: string, updates: Partial<User>) {
  const user = await this.userRepository.update(userId, updates);
  
  // Invalidate everywhere
  await this.redis.publish('invalidate', `user:${userId}`);
  
  return user;
}
```

### 3. Unbounded Cache Growth

Cache never evicts, memory bloats.

```typescript
// ✅ Configure eviction policy
CacheModule.register({
  isGlobal: true,
  store: redisStore,
  host: 'localhost',
  port: 6379,
  maxmemory: '256mb', // Set limit
  maxmemory_policy: 'allkeys-lru', // Evict LRU keys when full
})
```
