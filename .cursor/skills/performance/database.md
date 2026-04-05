# Database Query Optimization

This document covers database indexing, query patterns, and performance tuning.

## Indexing Strategy

### Index Types

#### 1. Single-Column Index

Optimizes queries filtering on one column.

```typescript
@Entity('users')
@Index('idx_email', ['email']) // Single column
@Index('idx_status', ['status'])
@Index('idx_created_at', ['createdAt'])
export class User {
  @PrimaryGeneratedColumn('uuid')
  id: string;

  @Column()
  email: string;

  @Column()
  status: string;

  @CreateDateColumn()
  createdAt: Date;
}

// These queries benefit from indexes
// SELECT * FROM users WHERE email = ?
// SELECT * FROM users WHERE status = 'active'
// SELECT * FROM users WHERE createdAt > ?
```

#### 2. Composite Index

Optimizes queries filtering on multiple columns.

```typescript
@Entity('orders')
@Index('idx_customer_status', ['customerId', 'status']) // Composite
@Index('idx_date_status', ['createdAt', 'status'])
export class Order {
  @PrimaryGeneratedColumn('uuid')
  id: string;

  @Column()
  customerId: string;

  @Column()
  status: string;

  @CreateDateColumn()
  createdAt: Date;

  @Column()
  total: number;
}

// Benefits from idx_customer_status
// SELECT * FROM orders WHERE customerId = ? AND status = 'pending'

// ✅ GOOD - Matches index order
const orders = await orderRepository
  .createQueryBuilder('o')
  .where('o.customerId = :customerId', { customerId })
  .andWhere('o.status = :status', { status: 'pending' })
  .getMany();

// ⚠️ LESS EFFECTIVE - Reversed order
const orders = await orderRepository
  .createQueryBuilder('o')
  .where('o.status = :status', { status: 'pending' })
  .andWhere('o.customerId = :customerId', { customerId })
  .getMany();
```

#### 3. Unique Index

Enforces uniqueness while improving query speed.

```typescript
@Entity('users')
@Index('idx_email_unique', ['email'], { unique: true })
export class User {
  @Column({ unique: true })
  email: string;
}

// Also optimizes lookups by email
```

#### 4. Full-Text Index

Optimizes text searching.

```typescript
@Entity('articles')
@Index('idx_title_body', ['title', 'body'], { fulltext: true })
export class Article {
  @Column()
  title: string;

  @Column('text')
  body: string;
}

// Full-text search query
const results = await articleRepository
  .createQueryBuilder('a')
  .where("MATCH(a.title, a.body) AGAINST(:query IN BOOLEAN MODE)", { 
    query: 'NestJS performance'
  })
  .getMany();
```

### Index Design Principles

#### 1. Index Only Frequently Queried Columns

```typescript
// ✅ Query patterns drive indexing
// - Find users by email: Index email
// - Find active users: Index status
// - Find recent signups: Index createdAt

@Entity('users')
@Index('idx_email', ['email'])
@Index('idx_status', ['status'])
@Index('idx_created_at', ['createdAt'])
export class User {
  id: string;
  email: string;
  status: string;
  createdAt: Date;
  bio: string; // Rarely queried - NO INDEX
}
```

#### 2. Use Composite Indexes for Common Filters

```typescript
// Common query: Find pending orders by customer
@Entity('orders')
@Index('idx_customer_status', ['customerId', 'status']) // One index
// Instead of two separate indexes:
// @Index('idx_customer', ['customerId'])
// @Index('idx_status', ['status'])
export class Order {
  customerId: string;
  status: string;
}
```

#### 3. Column Order in Composite Indexes

Columns should match query filter order (most selective first).

```typescript
// Query: Find active users created after date
// SELECT * FROM users WHERE createdAt > ? AND status = 'active'

// ✅ GOOD - Most selective first (status has fewer values)
@Index('idx_status_date', ['status', 'createdAt'])

// ⚠️ LESS EFFECTIVE - Date first (many values)
@Index('idx_date_status', ['createdAt', 'status'])
```

## Query Optimization Patterns

### 1. N+1 Query Prevention

#### Problem: N+1 Queries

```typescript
// ❌ BAD - N+1 queries
async getOrdersWithItems(): Promise<Order[]> {
  const orders = await this.orderRepository.find(); // 1 query

  const enriched = await Promise.all(
    orders.map(order =>
      this.orderRepository.find({
        where: { id: order.id },
        relations: ['items'], // 1 query per order = N queries
      })
    )
  );

  return enriched;
}
// Total: 1 + N queries

// With 1000 orders: 1001 queries!
```

#### Solution: Eager Loading

```typescript
// ✅ GOOD - Eager loading with relations
async getOrdersWithItems(): Promise<Order[]> {
  return this.orderRepository.find({
    relations: ['items', 'customer'],
    order: { createdAt: 'DESC' },
  });
  // Total: 1 query with JOIN
}

// Using QueryBuilder for more control
async getOrdersWithItems(): Promise<Order[]> {
  return this.orderRepository
    .createQueryBuilder('order')
    .leftJoinAndSelect('order.items', 'items')
    .leftJoinAndSelect('order.customer', 'customer')
    .orderBy('order.createdAt', 'DESC')
    .getMany();
}
```

#### Solution: Batch Loading

```typescript
// ✅ GOOD - For cases where eager loading isn't possible
async getOrdersWithItems(): Promise<Order[]> {
  const orders = await this.orderRepository.find();

  // Batch load all items at once
  const allItems = await this.itemRepository.find({
    where: { orderId: In(orders.map(o => o.id)) },
  });

  // Map items to orders
  const itemsByOrderId = groupBy(allItems, 'orderId');
  orders.forEach(order => {
    order.items = itemsByOrderId[order.id] || [];
  });

  return orders;
  // Total: 2 queries (not 1 + N)
}
```

### 2. Pagination for Large Result Sets

```typescript
// ❌ BAD - Fetches all records
async getAllUsers(): Promise<User[]> {
  return this.userRepository.find(); // Could be millions!
}

// ✅ GOOD - Pagination
async getUsersPaginated(page: number = 1, limit: number = 20): Promise<PaginatedResult<User>> {
  const skip = (page - 1) * limit;

  const [data, total] = await this.userRepository.findAndCount({
    skip,
    take: limit,
    order: { createdAt: 'DESC' },
  });

  return {
    data,
    total,
    page,
    limit,
    pages: Math.ceil(total / limit),
  };
}

// Cursor-based pagination for large offsets
async getUsersCursorPaginated(
  limit: number = 20,
  cursor?: string
): Promise<CursorPaginatedResult<User>> {
  const query = this.userRepository
    .createQueryBuilder('user')
    .orderBy('user.id', 'ASC')
    .take(limit + 1);

  if (cursor) {
    query.where('user.id > :cursor', { cursor });
  }

  const items = await query.getMany();
  const hasMore = items.length > limit;

  return {
    data: items.slice(0, limit),
    nextCursor: hasMore ? items[limit - 1].id : null,
    hasMore,
  };
}
```

### 3. Select Only Needed Columns

```typescript
// ❌ BAD - Fetches all columns
async getUserEmails(): Promise<User[]> {
  return this.userRepository.find();
  // Returns all fields (id, email, password, bio, profile, etc.)
}

// ✅ GOOD - Select specific columns
async getUserEmails(): Promise<{ id: string; email: string }[]> {
  return this.userRepository.find({
    select: { id: true, email: true }, // Only these columns
  });
}

// Using QueryBuilder
async getUserEmails(): Promise<{ id: string; email: string }[]> {
  return this.userRepository
    .createQueryBuilder('user')
    .select(['user.id', 'user.email'])
    .getMany();
}
```

### 4. Complex Queries with QueryBuilder

```typescript
@Injectable()
export class OrderRepository extends Repository<Order> {
  // Find high-value customers with recent orders
  async findHighValueCustomers(minOrderValue: number = 1000): Promise<any[]> {
    return this.createQueryBuilder('order')
      .select('customer.id', 'customerId')
      .addSelect('customer.name', 'customerName')
      .addSelect('COUNT(order.id)', 'orderCount')
      .addSelect('SUM(order.total)', 'totalSpent')
      .innerJoin('order.customer', 'customer')
      .where('order.createdAt > :date', { 
        date: new Date(Date.now() - 30 * 24 * 60 * 60 * 1000) // Last 30 days
      })
      .having('SUM(order.total) > :minValue', { minValue: minOrderValue })
      .groupBy('customer.id')
      .orderBy('totalSpent', 'DESC')
      .getRawMany();
  }

  // Find orders with filters and pagination
  async findFiltered(
    filters: OrderFilterDto,
    page: number = 1,
    limit: number = 20
  ): Promise<Order[]> {
    const query = this.createQueryBuilder('order')
      .leftJoinAndSelect('order.customer', 'customer')
      .leftJoinAndSelect('order.items', 'items');

    if (filters.customerId) {
      query.where('order.customerId = :customerId', { 
        customerId: filters.customerId 
      });
    }

    if (filters.status) {
      query.andWhere('order.status IN (:...statuses)', { 
        statuses: Array.isArray(filters.status) ? filters.status : [filters.status]
      });
    }

    if (filters.minTotal) {
      query.andWhere('order.total >= :minTotal', { 
        minTotal: filters.minTotal 
      });
    }

    if (filters.dateFrom) {
      query.andWhere('order.createdAt >= :dateFrom', { 
        dateFrom: filters.dateFrom 
      });
    }

    const skip = (page - 1) * limit;

    return query
      .orderBy('order.createdAt', 'DESC')
      .skip(skip)
      .take(limit)
      .getMany();
  }
}
```

## Query Execution Analysis

### Using EXPLAIN

Analyze query execution plan:

```sql
-- MySQL
EXPLAIN SELECT * FROM users WHERE status = 'active' AND createdAt > ?;

-- Shows:
-- - Which indexes are used
-- - Rows examined
-- - Estimated execution cost

EXPLAIN FORMAT=JSON SELECT ...;
-- Detailed JSON output
```

### Connection Pooling

```typescript
@Module({
  imports: [
    TypeOrmModule.forRoot({
      type: 'postgres',
      host: 'localhost',
      database: 'app',
      entities: [],
      // Connection pooling
      extra: {
        max: 20, // Max connections
        min: 5, // Min connections
        idleTimeoutMillis: 30000,
        connectionTimeoutMillis: 2000,
      },
    }),
  ],
})
export class DatabaseModule {}
```

## Performance Anti-Patterns

### 1. Missing Indexes

```typescript
// ❌ SLOW - Unindexed query on large table
// SELECT * FROM users WHERE status = 'active'
// Scans all rows!

// ✅ FAST - With index
@Index('idx_status', ['status'])
```

### 2. SELECT *

```typescript
// ❌ SLOW - Unnecessary columns
await userRepository.find();

// ✅ FAST - Only needed columns
await userRepository.find({
  select: { id: true, email: true },
});
```

### 3. Unbounded Queries

```typescript
// ❌ SLOW - No limit
await userRepository.find({ order: { createdAt: 'DESC' } });

// ✅ FAST - Limited
await userRepository.find({
  take: 100,
  order: { createdAt: 'DESC' },
});
```

### 4. Complex Joins

```typescript
// ❌ Slow joins
.leftJoinAndSelect('order.customer', 'customer')
.leftJoinAndSelect('order.items', 'items')
.leftJoinAndSelect('items.product', 'product')
.leftJoinAndSelect('product.category', 'category')
.leftJoinAndSelect('product.supplier', 'supplier')

// ✅ Better - Load only what's needed
// Or use separate queries with batching
```

### 5. Sorting Without Index

```typescript
// ❌ SLOW - No index on column being sorted
await orders.find({
  order: { estimatedDelivery: 'ASC' }, // No index
});

// ✅ FAST - Index exists
@Index('idx_estimated_delivery', ['estimatedDelivery'])
```

## Monitoring Query Performance

### Enable Query Logging

```typescript
TypeOrmModule.forRoot({
  logging: ['query', 'error'],
  logger: 'advanced-console', // or 'file', 'database'
  maxQueryExecutionTime: 1000, // Log queries > 1 second
})
```

### Query Analysis Service

```typescript
@Injectable()
export class QueryAnalysisService {
  async analyzeSlowQueries(): Promise<SlowQueryReport> {
    // Query MySQL slow log
    const slowQueries = await this.dataSource.query(`
      SELECT query, count, total_time
      FROM mysql.slow_log
      ORDER BY total_time DESC
      LIMIT 10
    `);

    return {
      queries: slowQueries,
      recommendations: this.generateRecommendations(slowQueries),
    };
  }

  private generateRecommendations(queries: any[]): string[] {
    return queries.map(q => {
      if (!q.query.includes('INDEX')) {
        return `Add index for: ${q.query}`;
      }
      return null;
    }).filter(Boolean);
  }
}
```
