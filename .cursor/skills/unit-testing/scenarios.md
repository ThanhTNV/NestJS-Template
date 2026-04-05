# Complex Testing Scenarios

This document covers testing patterns for complex, real-world scenarios.

## Scenario 1: Service with Multiple Dependencies

**Problem**: Service orchestrates multiple repositories and external services.

```typescript
@Injectable()
export class OrderService {
  constructor(
    private readonly orderRepository: OrderRepository,
    private readonly productService: ProductService,
    private readonly paymentService: PaymentService,
    private readonly notificationService: NotificationService,
    private readonly logger: Logger,
  ) {}

  async createOrder(dto: CreateOrderDto, userId: string): Promise<Order> {
    const product = await this.productService.getProduct(dto.productId);
    
    if (!product || product.stock < dto.quantity) {
      throw new OutOfStockError('Product out of stock');
    }

    const amount = product.price * dto.quantity;
    
    const payment = await this.paymentService.charge(userId, amount);
    if (!payment.success) {
      throw new PaymentFailedError('Payment declined');
    }

    const order = new Order();
    order.productId = dto.productId;
    order.quantity = dto.quantity;
    order.userId = userId;
    order.amount = amount;
    order.paymentId = payment.transactionId;

    const saved = await this.orderRepository.save(order);

    // Fire and forget - don't fail if notification fails
    this.notificationService.sendOrderConfirmation(saved).catch(err => {
      this.logger.warn('Failed to send confirmation', err);
    });

    return saved;
  }
}
```

**Test Setup:**

```typescript
describe('OrderService - Multiple Dependencies', () => {
  let service: OrderService;

  const mockOrderRepository = {
    save: jest.fn(),
    findById: jest.fn(),
  };

  const mockProductService = {
    getProduct: jest.fn(),
  };

  const mockPaymentService = {
    charge: jest.fn(),
  };

  const mockNotificationService = {
    sendOrderConfirmation: jest.fn(),
  };

  const mockLogger = {
    warn: jest.fn(),
    error: jest.fn(),
  };

  beforeEach(async () => {
    const module: TestingModule = await Test.createTestingModule({
      providers: [
        OrderService,
        {
          provide: OrderRepository,
          useValue: mockOrderRepository,
        },
        {
          provide: ProductService,
          useValue: mockProductService,
        },
        {
          provide: PaymentService,
          useValue: mockPaymentService,
        },
        {
          provide: NotificationService,
          useValue: mockNotificationService,
        },
        {
          provide: Logger,
          useValue: mockLogger,
        },
      ],
    }).compile();

    service = module.get<OrderService>(OrderService);
    jest.clearAllMocks();
  });

  describe('createOrder', () => {
    it('should create order with all dependencies', async () => {
      const dto: CreateOrderDto = {
        productId: '1',
        quantity: 2,
      };
      const userId = 'user_123';

      // Arrange
      mockProductService.getProduct.mockResolvedValue({
        id: '1',
        name: 'Product',
        price: 50,
        stock: 10,
      });

      mockPaymentService.charge.mockResolvedValue({
        success: true,
        transactionId: 'txn_123',
      });

      mockOrderRepository.save.mockResolvedValue({
        id: 'order_1',
        productId: '1',
        quantity: 2,
        userId,
        amount: 100,
        paymentId: 'txn_123',
      });

      mockNotificationService.sendOrderConfirmation.mockResolvedValue(undefined);

      // Act
      const result = await service.createOrder(dto, userId);

      // Assert - Verify all dependencies called correctly
      expect(mockProductService.getProduct).toHaveBeenCalledWith('1');
      expect(mockPaymentService.charge).toHaveBeenCalledWith(userId, 100);
      expect(mockOrderRepository.save).toHaveBeenCalled();
      expect(mockNotificationService.sendOrderConfirmation).toHaveBeenCalled();
      expect(result.id).toBe('order_1');
    });

    it('should reject if product out of stock', async () => {
      const dto: CreateOrderDto = {
        productId: '1',
        quantity: 100,
      };

      // Arrange
      mockProductService.getProduct.mockResolvedValue({
        id: '1',
        name: 'Product',
        stock: 5, // Less than requested
      });

      // Act & Assert
      await expect(service.createOrder(dto, 'user_123'))
        .rejects.toThrow(OutOfStockError);

      // Verify payment wasn't charged
      expect(mockPaymentService.charge).not.toHaveBeenCalled();
      expect(mockOrderRepository.save).not.toHaveBeenCalled();
    });

    it('should reject if payment fails', async () => {
      const dto: CreateOrderDto = {
        productId: '1',
        quantity: 2,
      };

      mockProductService.getProduct.mockResolvedValue({
        id: '1',
        name: 'Product',
        price: 50,
        stock: 10,
      });

      mockPaymentService.charge.mockResolvedValue({
        success: false,
        error: 'Declined',
      });

      // Act & Assert
      await expect(service.createOrder(dto, 'user_123'))
        .rejects.toThrow(PaymentFailedError);

      // Verify order wasn't created
      expect(mockOrderRepository.save).not.toHaveBeenCalled();
    });

    it('should handle notification failure gracefully', async () => {
      const dto: CreateOrderDto = {
        productId: '1',
        quantity: 2,
      };

      mockProductService.getProduct.mockResolvedValue({
        id: '1',
        name: 'Product',
        price: 50,
        stock: 10,
      });

      mockPaymentService.charge.mockResolvedValue({
        success: true,
        transactionId: 'txn_123',
      });

      mockOrderRepository.save.mockResolvedValue({
        id: 'order_1',
        productId: '1',
        quantity: 2,
        amount: 100,
        paymentId: 'txn_123',
      });

      // Notification fails
      mockNotificationService.sendOrderConfirmation.mockRejectedValue(
        new Error('Service down')
      );

      // Act
      const result = await service.createOrder(dto, 'user_123');

      // Assert - Order still created despite notification failure
      expect(result.id).toBe('order_1');
      expect(mockLogger.warn).toHaveBeenCalledWith(
        'Failed to send confirmation',
        expect.any(Error)
      );
    });
  });
});
```

## Scenario 2: Transaction & Rollback

**Problem**: Operation must succeed atomically or rollback entirely.

```typescript
@Injectable()
export class TransferService {
  constructor(
    private readonly accountRepository: AccountRepository,
    private readonly transactionRepository: TransactionRepository,
  ) {}

  async transfer(fromId: string, toId: string, amount: number): Promise<Transaction> {
    const fromAccount = await this.accountRepository.findById(fromId);
    if (!fromAccount) throw new NotFoundException('From account not found');

    const toAccount = await this.accountRepository.findById(toId);
    if (!toAccount) throw new NotFoundException('To account not found');

    if (fromAccount.balance < amount) {
      throw new InsufficientFundsError();
    }

    // Create transaction record
    const txn = new Transaction();
    txn.fromAccountId = fromId;
    txn.toAccountId = toId;
    txn.amount = amount;
    txn.status = TransactionStatus.PENDING;

    const savedTxn = await this.transactionRepository.save(txn);

    try {
      // Deduct from source
      await this.accountRepository.updateBalance(
        fromId,
        fromAccount.balance - amount
      );

      // Add to destination
      await this.accountRepository.updateBalance(
        toId,
        toAccount.balance + amount
      );

      // Mark transaction as complete
      savedTxn.status = TransactionStatus.COMPLETED;
      await this.transactionRepository.update(savedTxn.id, savedTxn);

      return savedTxn;
    } catch (error) {
      // Rollback transaction
      savedTxn.status = TransactionStatus.FAILED;
      await this.transactionRepository.update(savedTxn.id, savedTxn);
      throw error;
    }
  }
}
```

**Test Setup:**

```typescript
describe('TransferService - Transaction', () => {
  let service: TransferService;

  const mockAccountRepository = {
    findById: jest.fn(),
    updateBalance: jest.fn(),
  };

  const mockTransactionRepository = {
    save: jest.fn(),
    update: jest.fn(),
  };

  beforeEach(async () => {
    const module: TestingModule = await Test.createTestingModule({
      providers: [
        TransferService,
        {
          provide: AccountRepository,
          useValue: mockAccountRepository,
        },
        {
          provide: TransactionRepository,
          useValue: mockTransactionRepository,
        },
      ],
    }).compile();

    service = module.get<TransferService>(TransferService);
    jest.clearAllMocks();
  });

  it('should complete transfer', async () => {
    // Arrange
    mockAccountRepository.findById
      .mockResolvedValueOnce({ id: 'from', balance: 1000 })
      .mockResolvedValueOnce({ id: 'to', balance: 500 });

    mockTransactionRepository.save.mockResolvedValue({
      id: 'txn_1',
      status: TransactionStatus.PENDING,
    });

    mockAccountRepository.updateBalance.mockResolvedValue(undefined);

    mockTransactionRepository.update.mockResolvedValue({
      id: 'txn_1',
      status: TransactionStatus.COMPLETED,
    });

    // Act
    const result = await service.transfer('from', 'to', 100);

    // Assert
    expect(result.status).toBe(TransactionStatus.COMPLETED);
    expect(mockAccountRepository.updateBalance).toHaveBeenCalledTimes(2);
    expect(mockAccountRepository.updateBalance).toHaveBeenNthCalledWith(1, 'from', 900);
    expect(mockAccountRepository.updateBalance).toHaveBeenNthCalledWith(2, 'to', 600);
  });

  it('should rollback on failure', async () => {
    // Arrange
    mockAccountRepository.findById
      .mockResolvedValueOnce({ id: 'from', balance: 1000 })
      .mockResolvedValueOnce({ id: 'to', balance: 500 });

    mockTransactionRepository.save.mockResolvedValue({
      id: 'txn_1',
      status: TransactionStatus.PENDING,
    });

    // First update succeeds, second fails
    mockAccountRepository.updateBalance
      .mockResolvedValueOnce(undefined)
      .mockRejectedValueOnce(new Error('Database error'));

    // Act & Assert
    await expect(service.transfer('from', 'to', 100))
      .rejects.toThrow('Database error');

    // Verify rollback
    expect(mockTransactionRepository.update).toHaveBeenCalledWith(
      'txn_1',
      expect.objectContaining({ status: TransactionStatus.FAILED })
    );
  });

  it('should reject transfer without sufficient funds', async () => {
    mockAccountRepository.findById
      .mockResolvedValueOnce({ id: 'from', balance: 50 })
      .mockResolvedValueOnce({ id: 'to', balance: 500 });

    await expect(service.transfer('from', 'to', 100))
      .rejects.toThrow(InsufficientFundsError);

    expect(mockTransactionRepository.save).not.toHaveBeenCalled();
  });
});
```

## Scenario 3: Caching & Cache Invalidation

**Problem**: Service caches results and must invalidate on changes.

```typescript
@Injectable()
export class UserCacheService {
  private cache = new Map<string, User>();

  constructor(
    private readonly userRepository: UserRepository,
  ) {}

  async getUser(id: string): Promise<User> {
    // Check cache
    if (this.cache.has(id)) {
      return this.cache.get(id);
    }

    // Fetch from repository
    const user = await this.userRepository.findById(id);
    
    if (user) {
      this.cache.set(id, user);
    }

    return user;
  }

  async updateUser(id: string, updates: Partial<User>): Promise<User> {
    const user = await this.userRepository.findById(id);
    if (!user) throw new NotFoundException();

    Object.assign(user, updates);
    const updated = await this.userRepository.save(user);

    // Invalidate cache
    this.cache.delete(id);

    return updated;
  }

  clearCache(): void {
    this.cache.clear();
  }
}
```

**Test Setup:**

```typescript
describe('UserCacheService', () => {
  let service: UserCacheService;

  const mockUserRepository = {
    findById: jest.fn(),
    save: jest.fn(),
  };

  beforeEach(async () => {
    const module: TestingModule = await Test.createTestingModule({
      providers: [
        UserCacheService,
        {
          provide: UserRepository,
          useValue: mockUserRepository,
        },
      ],
    }).compile();

    service = module.get<UserCacheService>(UserCacheService);
    jest.clearAllMocks();
  });

  it('should return user from cache on second call', async () => {
    const user = { id: '1', name: 'John', email: 'john@test.com' };

    mockUserRepository.findById.mockResolvedValue(user);

    // First call - fetches from repository
    const result1 = await service.getUser('1');
    expect(result1).toEqual(user);
    expect(mockUserRepository.findById).toHaveBeenCalledTimes(1);

    // Second call - returns from cache
    const result2 = await service.getUser('1');
    expect(result2).toEqual(user);
    expect(mockUserRepository.findById).toHaveBeenCalledTimes(1); // Still 1!
  });

  it('should invalidate cache on update', async () => {
    const user = { id: '1', name: 'John', email: 'john@test.com' };
    const updated = { ...user, name: 'Jane' };

    // Prime cache
    mockUserRepository.findById.mockResolvedValueOnce(user);
    await service.getUser('1');

    // Update user
    mockUserRepository.findById.mockResolvedValueOnce(user);
    mockUserRepository.save.mockResolvedValue(updated);

    await service.updateUser('1', { name: 'Jane' });

    // Next fetch should hit repository again
    mockUserRepository.findById.mockResolvedValueOnce(updated);
    const result = await service.getUser('1');

    expect(result).toEqual(updated);
    expect(mockUserRepository.findById).toHaveBeenCalledTimes(3); // Primed + update lookup + re-fetch
  });

  it('should clear entire cache', async () => {
    const user1 = { id: '1', name: 'John' };
    const user2 = { id: '2', name: 'Jane' };

    mockUserRepository.findById
      .mockResolvedValueOnce(user1)
      .mockResolvedValueOnce(user2)
      .mockResolvedValueOnce(user1)
      .mockResolvedValueOnce(user2);

    // Prime cache with 2 users
    await service.getUser('1');
    await service.getUser('2');

    // Clear cache
    service.clearCache();

    // Next calls hit repository
    await service.getUser('1');
    await service.getUser('2');

    expect(mockUserRepository.findById).toHaveBeenCalledTimes(4);
  });
});
```

## Scenario 4: Event-Driven Architecture

**Problem**: Service publishes events when state changes.

```typescript
@Injectable()
export class UserService {
  constructor(
    private readonly userRepository: UserRepository,
    private readonly eventEmitter: EventEmitter2,
  ) {}

  async createUser(dto: CreateUserDto): Promise<User> {
    const user = new User();
    user.email = dto.email;
    user.name = dto.name;

    const saved = await this.userRepository.save(user);

    // Emit event
    this.eventEmitter.emit('user.created', {
      userId: saved.id,
      email: saved.email,
    });

    return saved;
  }

  async deleteUser(id: string): Promise<void> {
    const user = await this.userRepository.findById(id);
    if (!user) throw new NotFoundException();

    await this.userRepository.delete(id);

    // Emit event
    this.eventEmitter.emit('user.deleted', {
      userId: id,
      email: user.email,
    });
  }
}
```

**Test Setup:**

```typescript
describe('UserService - Events', () => {
  let service: UserService;

  const mockUserRepository = {
    save: jest.fn(),
    findById: jest.fn(),
    delete: jest.fn(),
  };

  const mockEventEmitter = {
    emit: jest.fn(),
  };

  beforeEach(async () => {
    const module: TestingModule = await Test.createTestingModule({
      providers: [
        UserService,
        {
          provide: UserRepository,
          useValue: mockUserRepository,
        },
        {
          provide: EventEmitter2,
          useValue: mockEventEmitter,
        },
      ],
    }).compile();

    service = module.get<UserService>(UserService);
    jest.clearAllMocks();
  });

  it('should emit user.created event', async () => {
    const dto: CreateUserDto = {
      email: 'test@test.com',
      name: 'John',
    };

    mockUserRepository.save.mockResolvedValue({
      id: '1',
      email: 'test@test.com',
      name: 'John',
    });

    await service.createUser(dto);

    expect(mockEventEmitter.emit).toHaveBeenCalledWith(
      'user.created',
      expect.objectContaining({
        userId: '1',
        email: 'test@test.com',
      })
    );
  });

  it('should emit user.deleted event', async () => {
    mockUserRepository.findById.mockResolvedValue({
      id: '1',
      email: 'test@test.com',
    });
    mockUserRepository.delete.mockResolvedValue(undefined);

    await service.deleteUser('1');

    expect(mockEventEmitter.emit).toHaveBeenCalledWith(
      'user.deleted',
      expect.objectContaining({
        userId: '1',
        email: 'test@test.com',
      })
    );
  });
});
```
