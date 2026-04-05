# Architecture Violation Patterns

This document catalogs common architectural violations, their consequences, and how to fix them.

## Violation Categories

### Category 1: Controllers with Business Logic

**Problem:** Controllers make decisions that should be in services.

**Why it's bad:**
- Business logic is untestable (tied to HTTP)
- Logic can't be reused from other entry points (CLI, jobs, events)
- Controllers become thick and hard to maintain
- Violates Single Responsibility Principle

**Example:**
```typescript
// ❌ BAD
@Controller('users')
export class UserController {
  constructor(private readonly userRepository: UserRepository) {}

  @Post()
  async create(@Body() dto: CreateUserDto) {
    // Business logic in controller
    const existing = await this.userRepository.findByEmail(dto.email);
    if (existing) {
      throw new ConflictException('Email exists');
    }

    const user = new User();
    user.email = dto.email;
    user.name = dto.name;

    // Validation logic
    if (dto.age < 18) {
      throw new BadRequestException('Must be 18+');
    }

    return this.userRepository.save(user);
  }
}

// ✅ GOOD
@Controller('users')
export class UserController {
  constructor(private readonly userService: UserService) {}

  @Post()
  async create(@Body() dto: CreateUserDto) {
    return this.userService.createUser(dto);
  }
}

@Injectable()
export class UserService {
  constructor(private readonly userRepository: UserRepository) {}

  async createUser(dto: CreateUserDto): Promise<User> {
    const existing = await this.userRepository.findByEmail(dto.email);
    if (existing) {
      throw new ConflictException('Email exists');
    }

    this.validateAge(dto.age);

    const user = new User();
    user.email = dto.email;
    user.name = dto.name;
    user.age = dto.age;

    return this.userRepository.save(user);
  }

  private validateAge(age: number): void {
    if (age < 18) {
      throw new BadRequestException('Must be 18+');
    }
  }
}
```

**Testing impact:**
```typescript
// ❌ Untestable - requires HTTP context
describe('UserController', () => {
  it('should reject duplicate email', async () => {
    // Must set up full HTTP test environment
    const module = await Test.createTestingModule({
      controllers: [UserController],
      // ...
    }).compile();
    // Complex setup for business logic test
  });
});

// ✅ Testable - pure function
describe('UserService', () => {
  it('should reject duplicate email', async () => {
    const mockRepository = {
      findByEmail: jest.fn().mockResolvedValue({ id: '1' }),
    };
    const service = new UserService(mockRepository);

    await expect(
      service.createUser({ email: 'test@test.com' })
    ).rejects.toThrow(ConflictException);
  });
});
```

---

### Category 2: Repositories with Business Logic

**Problem:** Repositories make business decisions instead of just persisting data.

**Why it's bad:**
- Multiple services might need same logic (duplication)
- Business rules hidden in data layer (hard to audit)
- Repositories become stateful and complex
- Hard to implement alternative persistence strategies

**Example:**
```typescript
// ❌ BAD - Repository has business rules
@Injectable()
export class AccountRepository {
  async withdraw(accountId: string, amount: number): Promise<void> {
    const account = await this.db.query(
      'SELECT balance FROM accounts WHERE id = ?',
      [accountId]
    );

    // Business logic in repository
    if (account.balance < amount) {
      throw new InsufficientFundsError();
    }

    if (amount > 10000) {
      // Approval workflow
      await this.approvalService.requestApproval(accountId, amount);
    }

    await this.db.query(
      'UPDATE accounts SET balance = balance - ? WHERE id = ?',
      [amount, accountId]
    );
  }
}

// ✅ GOOD - Repository only persists
@Injectable()
export class AccountRepository {
  async findById(id: string): Promise<Account> {
    const record = await this.db.query(
      'SELECT * FROM accounts WHERE id = ?',
      [id]
    );
    return this.mapToDomain(record);
  }

  async updateBalance(id: string, newBalance: number): Promise<void> {
    await this.db.query(
      'UPDATE accounts SET balance = ? WHERE id = ?',
      [newBalance, id]
    );
  }

  private mapToDomain(record: any): Account {
    const account = new Account();
    account.id = record.id;
    account.balance = record.balance;
    return account;
  }
}

// Business logic in service
@Injectable()
export class AccountService {
  constructor(
    private readonly accountRepository: AccountRepository,
    private readonly approvalService: ApprovalService,
  ) {}

  async withdraw(accountId: string, amount: number): Promise<void> {
    const account = await this.accountRepository.findById(accountId);

    if (account.balance < amount) {
      throw new InsufficientFundsError();
    }

    if (amount > 10000) {
      await this.approvalService.requestApproval(accountId, amount);
    }

    const newBalance = account.balance - amount;
    await this.accountRepository.updateBalance(accountId, newBalance);
  }
}
```

---

### Category 3: Services with HTTP Concerns

**Problem:** Services depend on HTTP abstractions (status codes, exceptions, etc.).

**Why it's bad:**
- Service can't be used from non-HTTP entry points
- Tight coupling to NestJS HTTP module
- Business logic mixed with framework concerns
- Harder to migrate frameworks later

**Example:**
```typescript
// ❌ BAD - HTTP concerns in service
@Injectable()
export class ProductService {
  constructor(private readonly productRepository: ProductRepository) {}

  async getProduct(id: string): Promise<HttpResponse> {
    try {
      const product = await this.productRepository.findById(id);

      if (!product) {
        throw new HttpException(
          { error: 'Product not found', code: 'NOT_FOUND' },
          HttpStatus.NOT_FOUND,
        );
      }

      return {
        statusCode: 200,
        data: product,
        timestamp: new Date(),
      };
    } catch (error) {
      throw new HttpException(
        { error: error.message },
        HttpStatus.INTERNAL_SERVER_ERROR,
      );
    }
  }
}

// ✅ GOOD - Pure business logic in service
@Injectable()
export class ProductService {
  constructor(private readonly productRepository: ProductRepository) {}

  async getProduct(id: string): Promise<Product> {
    const product = await this.productRepository.findById(id);
    if (!product) {
      throw new ProductNotFoundError(id);
    }
    return product;
  }
}

// HTTP mapping in controller
@Controller('products')
export class ProductController {
  constructor(private readonly productService: ProductService) {}

  @Get(':id')
  async getProduct(@Param('id') id: string): Promise<Product> {
    return this.productService.getProduct(id);
    // NestJS automatically handles 200 response
  }
}

// Can also use from CLI or other entry points
@Injectable()
export class ProductExportJob {
  constructor(private readonly productService: ProductService) {}

  async run(): Promise<void> {
    const products = await this.productService.getAllProducts();
    // Use pure business logic without HTTP
    await this.exportToFile(products);
  }
}
```

---

### Category 4: Upward Dependencies (Anti-pattern)

**Problem:** Lower layers depend on higher layers (inverts dependency flow).

**Why it's bad:**
- Creates circular dependencies
- Violates dependency inversion principle
- Hard to test (dependencies on HTTP layer)
- Creates tight coupling

**Example:**
```typescript
// ❌ BAD - Repository depends on service
@Injectable()
export class UserRepository {
  constructor(
    private readonly auditService: AuditService, // SERVICE
    private readonly notificationService: NotificationService, // SERVICE
  ) {}

  async save(user: User): Promise<User> {
    const saved = await this.db.query('INSERT INTO users...', user);
    
    // VIOLATES - Repository calling services
    await this.auditService.log('user_created', saved.id);
    await this.notificationService.sendWelcome(saved.email);
    
    return saved;
  }
}

// ✅ GOOD - Service orchestrates
@Injectable()
export class UserService {
  constructor(
    private readonly userRepository: UserRepository,
    private readonly auditService: AuditService,
    private readonly notificationService: NotificationService,
  ) {}

  async createUser(dto: CreateUserDto): Promise<User> {
    const user = new User(dto);
    const saved = await this.userRepository.save(user);
    
    // Service orchestrates
    await this.auditService.log('user_created', saved.id);
    await this.notificationService.sendWelcome(saved.email);
    
    return saved;
  }
}
```

---

### Category 5: Direct Repository Access from Controller

**Problem:** Controllers bypass services and access repositories directly.

**Why it's bad:**
- Business logic scattered across controllers
- Can't reuse logic between controllers
- Violates layering
- Hard to test business logic

**Example:**
```typescript
// ❌ BAD - Direct repository access
@Controller('orders')
export class OrderController {
  constructor(
    private readonly orderRepository: OrderRepository,
    private readonly productRepository: ProductRepository,
  ) {}

  @Post()
  async createOrder(@Body() dto: CreateOrderDto): Promise<Order> {
    // Business logic scattered in controller
    const product = await this.productRepository.findById(dto.productId);
    
    if (!product) {
      throw new NotFoundException('Product not found');
    }

    if (product.stock < dto.quantity) {
      throw new BadRequestException('Insufficient stock');
    }

    const order = new Order();
    order.productId = dto.productId;
    order.quantity = dto.quantity;
    order.totalPrice = product.price * dto.quantity;

    // Update stock directly
    await this.productRepository.updateStock(
      dto.productId,
      product.stock - dto.quantity,
    );

    return this.orderRepository.save(order);
  }
}

// ✅ GOOD - Use service for business logic
@Controller('orders')
export class OrderController {
  constructor(private readonly orderService: OrderService) {}

  @Post()
  async createOrder(@Body() dto: CreateOrderDto): Promise<Order> {
    return this.orderService.createOrder(dto);
  }
}

@Injectable()
export class OrderService {
  constructor(
    private readonly orderRepository: OrderRepository,
    private readonly productService: ProductService,
  ) {}

  async createOrder(dto: CreateOrderDto): Promise<Order> {
    const product = await this.productService.getProduct(dto.productId);
    
    if (product.stock < dto.quantity) {
      throw new InsufficientStockError();
    }

    const order = new Order();
    order.productId = dto.productId;
    order.quantity = dto.quantity;
    order.totalPrice = product.price * dto.quantity;

    const saved = await this.orderRepository.save(order);
    
    // Update stock through service
    await this.productService.decreaseStock(dto.productId, dto.quantity);
    
    return saved;
  }
}
```

---

### Category 6: Circular Dependencies

**Problem:** Module A depends on B, B depends on A.

**Why it's bad:**
- Breaks module isolation
- Makes tree-shaking impossible
- Hard to test in isolation
- Unclear data flow

**Example:**
```typescript
// ❌ BAD - Circular dependency
@Injectable()
export class UserService {
  constructor(private readonly authService: AuthService) {}

  async createUser(dto: CreateUserDto): Promise<User> {
    // Check permissions through auth service
    if (!await this.authService.hasPermission('create_user')) {
      throw new ForbiddenException();
    }
    // ...
  }
}

@Injectable()
export class AuthService {
  constructor(private readonly userService: UserService) {}

  async hasPermission(permission: string): Promise<boolean> {
    // Look up user through user service
    const user = await this.userService.getUser(this.getCurrentUserId());
    return user.permissions.includes(permission);
  }
}

// ✅ GOOD - Extract shared concern
@Injectable()
export class PermissionService {
  // No dependencies on User or Auth services
  hasPermission(userPermissions: string[], required: string): boolean {
    return userPermissions.includes(required);
  }
}

@Injectable()
export class UserService {
  constructor(private readonly permissionService: PermissionService) {}

  async createUser(dto: CreateUserDto, userPermissions: string[]): Promise<User> {
    if (!this.permissionService.hasPermission(userPermissions, 'create_user')) {
      throw new ForbiddenException();
    }
    // ...
  }
}

@Injectable()
export class AuthService {
  constructor(private readonly permissionService: PermissionService) {}

  async hasPermission(permission: string): Promise<boolean> {
    const user = this.getCurrentUser();
    return this.permissionService.hasPermission(user.permissions, permission);
  }
}
```

---

### Category 7: Exposing Internal Details

**Problem:** Modules export internal implementation classes instead of public interfaces.

**Why it's bad:**
- Consumers depend on implementation details
- Internal refactoring breaks consumers
- Unclear public API
- Hard to mock/test

**Example:**
```typescript
// ❌ BAD - Exposing internals
@Module({
  providers: [UserService, UserRepository, UserFactory],
  exports: [UserService, UserRepository, UserFactory], // Too much!
})
export class UserModule {}

@Injectable()
export class OrderService {
  constructor(
    private readonly userRepository: UserRepository, // Internal detail!
    private readonly userFactory: UserFactory, // Internal detail!
  ) {}
}

// ✅ GOOD - Expose only public service
@Module({
  providers: [UserService, UserRepository, UserFactory],
  exports: [UserService], // Only what others need
})
export class UserModule {}

@Injectable()
export class OrderService {
  constructor(
    private readonly userService: UserService, // Public interface
  ) {}
}
```

---

## Anti-Pattern Detection Checklist

When reviewing code, check for:

- [ ] Any `@Post()` / `@Get()` calling repository directly
- [ ] Any `@Injectable()` service accepting multiple repositories
- [ ] Any repository method with business logic or validation
- [ ] Any service throwing `HttpException` or using `HttpStatus`
- [ ] Any upward dependency (repository → service, or service → controller)
- [ ] Module exporting more than services/controllers
- [ ] Shared modules that other modules depend on (creates tight coupling)
- [ ] Service A and Service B importing each other
- [ ] Business rules in DTOs beyond validation decorators
- [ ] Direct database calls outside repositories
