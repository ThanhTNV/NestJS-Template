# Feature Architecture & Design Patterns

This document provides architecture guidance for features during the delivery workflow.

## Layered Architecture Review

Before implementing, validate the architectural design:

### Domain Layer (Business Logic)

**Question**: What is the core business problem we're solving?

```typescript
// Example: Order Processing
Domain Logic:
- Orders have items, customer, total price
- Inventory decrements when order created
- Payment must succeed before confirming order
- Order states: pending → processing → shipped → delivered
```

**Design Checklist**:
- [ ] Core entities identified (Order, OrderItem, etc.)
- [ ] Business rules documented (state transitions, validations)
- [ ] Edge cases identified (what if payment fails?)
- [ ] External dependencies listed (payment provider, inventory system)

### Infrastructure Layer (Data & External Services)

**Question**: How do we persist and access data?

```typescript
// Pattern: Repository Pattern
OrdersRepository {
  findById(id): Order
  findByUserId(userId): Order[]
  create(order): Order
  update(id, updates): Order
  findPendingOrders(): Order[]  // Custom query
}

// External Services
PaymentService {
  charge(amount, token): PaymentResult
}
```

**Design Checklist**:
- [ ] Repository methods identified
- [ ] Database queries optimized (indexes, eager loading)
- [ ] External service contracts defined
- [ ] Error handling for external failures

### Application Layer (Orchestration)

**Question**: How do services coordinate?

```typescript
// Service Orchestration
OrderService {
  async createOrder(userId, items) {
    // 1. Validate
    await this.validateInventory(items);
    
    // 2. Create
    const order = await this.ordersRepository.create(...);
    
    // 3. External call
    await this.paymentsQueue.add('process', { orderId });
    
    // 4. Notify
    await this.notificationService.send(...);
    
    // 5. Return
    return order;
  }
}
```

**Design Checklist**:
- [ ] Service responsibilities clear (what this service owns)
- [ ] Dependencies injected (not created)
- [ ] External calls wrapped (error handling)
- [ ] Transactions considered (atomicity)
- [ ] Async operations queued where appropriate

### Presentation Layer (API)

**Question**: What does the client see?

```typescript
// Controller: Handles HTTP
OrdersController {
  @Post()
  async create(@Body() dto: CreateOrderDto) {
    return this.ordersService.createOrder(...);
  }
}

// DTO: Contract
CreateOrderDto {
  items: OrderItemDto[]
  shippingAddress: AddressDto
}

// Response: What client gets
OrderResponseDto {
  id: string
  total: number
  status: 'pending' | 'processing'
  // Don't expose: paymentMethodId, internalNotes
}
```

**Design Checklist**:
- [ ] Endpoints clear and RESTful
- [ ] DTOs match Swagger documentation
- [ ] Security checks applied (guards)
- [ ] Sensitive data removed from responses
- [ ] Input validation applied

## Common Patterns

### 1. Service with Multiple Dependencies

```typescript
@Injectable()
export class OrderService {
  constructor(
    private ordersRepository: OrdersRepository,
    private inventoryService: InventoryService,
    private paymentsQueue: Queue,
    private notificationService: NotificationService,
    @Inject(CACHE_MANAGER) private cache: Cache,
    private logger: Logger,
  ) {}

  async createOrder(userId: string, dto: CreateOrderDto): Promise<Order> {
    try {
      // 1. Validate using external service
      const availability = await this.inventoryService.checkAvailability(dto.items);
      if (!availability.allAvailable) {
        throw new OutOfStockError('Some items unavailable');
      }

      // 2. Create in database
      const order = await this.ordersRepository.create({
        userId,
        items: dto.items,
        total: this.calculateTotal(dto.items),
      });

      // 3. Queue heavy operation
      await this.paymentsQueue.add('process-payment', {
        orderId: order.id,
        amount: order.total,
      });

      // 4. Cache result briefly
      await this.cache.set(`order:${order.id}`, order, 300);

      // 5. Log success
      this.logger.log(`Order created: ${order.id}`);

      return order;
    } catch (error) {
      this.logger.error(`Order creation failed: ${error.message}`);
      throw error;
    }
  }
}
```

**Patterns Used**:
- Dependency injection
- Error handling with custom exceptions
- Logging at key points
- Caching for performance
- Queuing for async operations
- Clear try-catch with rethrow

### 2. Query-Heavy Service

```typescript
// Problem: Many different query patterns
// Solution: Use QueryBuilder for flexibility

@Injectable()
export class ReportService {
  async getOrderMetrics(filters: OrderFiltersDto): Promise<OrderMetrics> {
    return this.ordersRepository
      .createQueryBuilder('order')
      .select('DATE(order.createdAt)', 'date')
      .addSelect('COUNT(order.id)', 'orderCount')
      .addSelect('SUM(order.total)', 'revenue')
      .where('order.createdAt >= :startDate', { startDate: filters.startDate })
      .andWhere('order.status = :status', { status: 'completed' })
      .groupBy('DATE(order.createdAt)')
      .orderBy('date', 'DESC')
      .getRawMany();
  }

  async getTopCustomers(limit: number = 10): Promise<CustomerMetrics[]> {
    return this.ordersRepository
      .createQueryBuilder('order')
      .select('order.userId', 'userId')
      .addSelect('COUNT(order.id)', 'orderCount')
      .addSelect('SUM(order.total)', 'totalSpent')
      .groupBy('order.userId')
      .orderBy('totalSpent', 'DESC')
      .limit(limit)
      .getRawMany();
  }
}
```

**Patterns Used**:
- QueryBuilder for complex queries
- Aggregation queries
- Clear query structure
- Parameterized queries (prevent injection)

### 3. Transaction Handling

```typescript
// For operations that must succeed together

@Injectable()
export class OrderService {
  constructor(
    private dataSource: DataSource,
    private ordersRepository: OrdersRepository,
    private inventoryRepository: InventoryRepository,
  ) {}

  async createOrderWithInventoryUpdate(
    userId: string,
    items: OrderItemDto[],
  ): Promise<Order> {
    const queryRunner = this.dataSource.createQueryRunner();
    await queryRunner.connect();
    await queryRunner.startTransaction();

    try {
      // 1. Create order (in transaction)
      const order = await queryRunner.manager.save(Order, {
        userId,
        items,
      });

      // 2. Update inventory (in transaction)
      for (const item of items) {
        await queryRunner.manager.decrement(
          Inventory,
          { productId: item.productId },
          'quantity',
          item.quantity,
        );
      }

      // 3. Commit if both succeed
      await queryRunner.commitTransaction();
      return order;
    } catch (error) {
      // 4. Rollback if either fails
      await queryRunner.rollbackTransaction();
      throw new OrderCreationFailedError('Transaction rolled back');
    } finally {
      await queryRunner.release();
    }
  }
}
```

**Patterns Used**:
- Transaction wrapper
- Rollback on failure
- Atomic operations
- Error handling

### 4. Caching Strategy

```typescript
@Injectable()
export class ProductService {
  constructor(
    private productRepository: ProductRepository,
    @Inject(CACHE_MANAGER) private cache: Cache,
  ) {}

  // Read-through cache
  async getProduct(id: string): Promise<Product> {
    const cached = await this.cache.get<Product>(`product:${id}`);
    if (cached) return cached;

    const product = await this.productRepository.findById(id);
    if (product) {
      await this.cache.set(`product:${id}`, product, 3600); // 1 hour
    }
    return product;
  }

  // Cache invalidation on update
  async updateProduct(id: string, updates: Partial<Product>): Promise<Product> {
    const product = await this.productRepository.update(id, updates);
    
    // Invalidate cache
    await this.cache.del(`product:${id}`);
    
    // Invalidate related caches
    await this.cache.del(`products:list`);
    await this.cache.del(`products:category:${product.categoryId}`);

    return product;
  }

  // Cache warming
  async warmCache(): Promise<void> {
    const products = await this.productRepository.find();
    for (const product of products) {
      await this.cache.set(`product:${product.id}`, product, 3600);
    }
  }
}
```

**Patterns Used**:
- Read-through cache
- Cache invalidation
- Cache warming
- Proper TTL

### 5. Event-Driven Architecture

```typescript
// For decoupled services

@Injectable()
export class OrderService {
  constructor(
    private eventEmitter: EventEmitter2,
    private ordersRepository: OrdersRepository,
  ) {}

  async createOrder(userId: string, dto: CreateOrderDto): Promise<Order> {
    const order = await this.ordersRepository.create({
      userId,
      ...dto,
    });

    // Emit event (other services listen)
    this.eventEmitter.emit('order.created', {
      orderId: order.id,
      userId: order.userId,
      total: order.total,
    });

    return order;
  }
}

// Another service listens
@Injectable()
export class NotificationService {
  @OnEvent('order.created')
  async handleOrderCreated(payload: OrderCreatedEvent) {
    // Send notification
    await this.emailService.send(payload.userId, 'Order confirmation');
  }
}
```

**Patterns Used**:
- Event emission
- Loose coupling
- Event listeners
- Async operations

## Anti-Patterns to Avoid

### 1. Business Logic in Controller

```typescript
// ❌ BAD
@Controller('orders')
export class OrdersController {
  @Post()
  async create(@Body() dto: CreateOrderDto) {
    // Business logic in controller!
    const inventory = await this.db.query(`
      SELECT quantity FROM inventory WHERE product_id = ?
    `, [dto.productId]);

    if (inventory[0].quantity < dto.quantity) {
      throw new Error('Out of stock');
    }

    await this.db.query(`INSERT INTO orders ...`);
    return { success: true };
  }
}

// ✅ GOOD
@Controller('orders')
export class OrdersController {
  constructor(private ordersService: OrdersService) {}

  @Post()
  async create(@Body() dto: CreateOrderDto): Promise<OrderResponseDto> {
    return this.ordersService.createOrder(dto);
  }
}

@Injectable()
export class OrdersService {
  async createOrder(dto: CreateOrderDto): Promise<Order> {
    // Business logic here
    await this.validateInventory(dto);
    return this.ordersRepository.create(dto);
  }
}
```

### 2. Direct Database Calls

```typescript
// ❌ BAD
this.dataSource.query('SELECT * FROM orders WHERE user_id = ?', [userId]);

// ✅ GOOD
this.ordersRepository.findByUserId(userId);
```

### 3. Tight Coupling to External Services

```typescript
// ❌ BAD - Directly calling external API
async processPayment(amount: number) {
  const response = await this.httpClient.post(
    'https://stripe.com/v1/charges',
    { amount }
  );
}

// ✅ GOOD - Abstract behind service
async processPayment(amount: number) {
  return this.paymentService.charge(amount);
}

// If Stripe fails, we can easily swap for another provider
@Injectable()
export class StripePaymentService implements PaymentService {
  async charge(amount: number): Promise<PaymentResult> {
    // Stripe implementation
  }
}
```

### 4. Mixing Async and Sync Operations

```typescript
// ❌ BAD - Waits for email to send
async createOrder(dto: CreateOrderDto) {
  const order = await this.ordersRepository.create(dto);
  await this.emailService.send(order.customerId); // Slow!
  return order;
}

// ✅ GOOD - Queue email for async processing
async createOrder(dto: CreateOrderDto) {
  const order = await this.ordersRepository.create(dto);
  await this.notificationQueue.add('send-confirmation', { orderId: order.id });
  return order; // Fast response
}
```

## Design Decision Framework

When designing a feature, ask:

1. **What is the core responsibility?**
   - Should this be in service, repository, or external service?

2. **What data does it need?**
   - Single query or multiple? Indexed properly?
   - Need caching?

3. **What could fail?**
   - External services unavailable?
   - Database constraints?
   - Plan error handling

4. **How long does it take?**
   - <100ms: OK in request/response
   - >100ms: Consider queue or cache

5. **Who needs to know?**
   - Should other services be notified?
   - Events or direct coupling?
