---
name: architecture
description: Validate adherence to layered and hexagonal architecture patterns. Prevent mixing of concerns and enforce clear separation between controllers, services, repositories, and domain layers. Use when reviewing code structure, validating module boundaries, preventing architectural violations, or generating new features.
---

# Architecture

This skill ensures your NestJS project maintains clean layered/hexagonal architecture with clear separation of concerns and boundaries. Use it to audit existing code and guide generation of new features.

## Architecture Patterns

### Layered Architecture

The project uses a **4-layer architecture**:

```
┌─────────────────────────┐
│  Presentation Layer     │  Controllers - HTTP handling
│  (API Boundary)         │
├─────────────────────────┤
│  Application Layer      │  Services - Business logic
│  (Business Logic)       │  Orchestration, validation
├─────────────────────────┤
│  Domain Layer           │  Entities - Domain models
│  (Domain Models)        │  Business rules
├─────────────────────────┤
│  Infrastructure Layer   │  Repositories - Persistence
│  (Data Access)          │  Database, external services
└─────────────────────────┘
```

### Hexagonal Architecture (Ports & Adapters)

Each feature module acts as a hexagon:

```
         ┌──────────────────────┐
         │   Feature Module     │
         │   (Domain Core)      │
         │                      │
    ┌────┤  Business Logic      ├────┐
    │    │                      │    │
    │    └──────────────────────┘    │
    │           ▲                     │
    │       (Inbound Port)        (Outbound Port)
    │           │                     │
   HTTP      Service            Repository
  Adapter     Layer              Adapter
    │           │                     │
    └────────────▼─────────────────────┘
```

## Layer Responsibilities

### Presentation Layer (Controllers)

**Responsibilities:**
- Parse HTTP requests
- Validate request structure (via DTOs)
- Call appropriate service methods
- Format responses
- Handle HTTP-specific concerns (headers, status codes)

**Allowed:**
- HTTP decorators (`@Get`, `@Post`, `@Body`, `@Param`)
- Dependency injection of services
- Input validation (via ValidationPipe)
- Request/response formatting

**Forbidden:**
- Business logic
- Database queries
- Calling repositories directly
- Caching logic
- External API calls

### Application Layer (Services)

**Responsibilities:**
- Implement business logic
- Orchestrate repositories and domain objects
- Perform validations
- Handle transactions
- Coordinate with external services

**Allowed:**
- Repository method calls
- Domain logic and rules
- Data transformations
- External API integrations
- Caching strategies

**Forbidden:**
- HTTP handling
- Direct database queries
- Repository implementation details
- HTTP-specific code
- Request/response objects

### Domain Layer (Entities)

**Responsibilities:**
- Define domain models
- Encapsulate business rules
- Provide pure domain logic
- Document core concepts

**Allowed:**
- Properties and methods
- Getters and setters
- Pure domain calculations
- Enum definitions
- Value objects

**Forbidden:**
- Decorators (except domain-specific ones)
- Persistence concerns
- HTTP concerns
- External dependencies
- Business orchestration

### Infrastructure Layer (Repositories)

**Responsibilities:**
- Execute database queries
- Implement persistence strategy
- Handle data mapping (entities ↔ models)
- Provide data access abstractions

**Allowed:**
- Database queries
- ORM usage (TypeORM, Prisma, etc.)
- Data mapping logic
- Connection management
- Query optimization

**Forbidden:**
- Business logic
- HTTP concerns
- Service orchestration
- External API calls (unless data-fetching is the concern)
- Validation logic

## Boundary Rules

### Cross-Layer Communication

```
✅ ALLOWED DIRECTIONS (Dependencies flow downward)

Controllers
    ↓
Services
    ↓
Repositories
    ↓
Database

❌ FORBIDDEN DIRECTIONS (Never flow upward)

Database ↗ Repositories ↗ Services ↗ Controllers
```

### Within-Layer Communication

```
✅ ALLOWED
- Services calling other services (same layer)
- Repositories calling other repositories (same layer)
- Controllers delegating to each other (rare)

⚠️  CAREFUL
- Service A calling Service B: OK, but avoid circular dependencies
- Repository A calling Repository B: OK if aggregating data

❌ FORBIDDEN
- Service calling controller
- Repository calling service
- Any upward dependency
```

## Violation Patterns & Fixes

### ❌ Pattern 1: Database Query in Controller

```typescript
// BAD - Business logic in controller
@Controller('users')
export class UserController {
  @Post()
  async create(@Body() dto: CreateUserDto) {
    const user = new User();
    user.name = dto.name;
    // Direct database call - VIOLATES ARCHITECTURE
    const saved = await this.db.query('INSERT INTO users...', user);
    return saved;
  }
}

// ✅ GOOD - Delegate to service
@Controller('users')
export class UserController {
  constructor(private readonly userService: UserService) {}

  @Post()
  async create(@Body() dto: CreateUserDto) {
    return this.userService.create(dto);
  }
}
```

### ❌ Pattern 2: Business Logic in Repository

```typescript
// BAD - Validation in repository
@Injectable()
export class UserRepository {
  async save(user: User): Promise<User> {
    // Business rule validation - VIOLATES ARCHITECTURE
    if (user.age < 18 && user.age > 65) {
      throw new Error('Invalid age range');
    }
    return this.db.query('INSERT INTO users...', user);
  }
}

// ✅ GOOD - Repository only persists
@Injectable()
export class UserRepository {
  async save(user: User): Promise<User> {
    return this.db.query('INSERT INTO users...', user);
  }
}

// Business logic in service
@Injectable()
export class UserService {
  async create(dto: CreateUserDto): Promise<User> {
    this.validateAge(dto.age);
    const user = new User(dto);
    return this.userRepository.save(user);
  }

  private validateAge(age: number): void {
    if (age < 18 || age > 65) {
      throw new InvalidAgeError('Invalid age range');
    }
  }
}
```

### ❌ Pattern 3: HTTP Concerns in Service

```typescript
// BAD - HTTP in service
@Injectable()
export class UserService {
  async getUser(id: string): Promise<HttpResponse> {
    // HTTP concerns in service - VIOLATES ARCHITECTURE
    const user = await this.userRepository.findById(id);
    if (!user) {
      throw new HttpException('Not found', HttpStatus.NOT_FOUND);
    }
    return { statusCode: 200, data: user };
  }
}

// ✅ GOOD - Service returns domain objects
@Injectable()
export class UserService {
  async getUser(id: string): Promise<User> {
    return this.userRepository.findById(id);
  }
}

// Controller handles HTTP
@Controller('users')
export class UserController {
  @Get(':id')
  async getUser(@Param('id') id: string): Promise<User> {
    const user = await this.userService.getUser(id);
    if (!user) {
      throw new NotFoundException('User not found');
    }
    return user;
  }
}
```

### ❌ Pattern 4: Circular Dependencies

```typescript
// BAD - Circular dependency
@Injectable()
export class UserService {
  constructor(private readonly authService: AuthService) {}
  
  async create(dto: CreateUserDto): Promise<User> {
    // Calls auth service
    await this.authService.validateCredentials(dto);
    return this.userRepository.save(new User(dto));
  }
}

@Injectable()
export class AuthService {
  constructor(private readonly userService: UserService) {}
  
  async validateCredentials(dto: any): Promise<void> {
    // Calls user service - CIRCULAR DEPENDENCY
    const user = await this.userService.findByEmail(dto.email);
    // ...
  }
}

// ✅ GOOD - Extract shared logic to shared service
@Injectable()
export class CredentialService {
  // Neutral service used by both
  validatePassword(plain: string, hash: string): boolean {
    // Pure validation logic
  }
}

@Injectable()
export class UserService {
  constructor(private readonly credentialService: CredentialService) {}
}

@Injectable()
export class AuthService {
  constructor(private readonly credentialService: CredentialService) {}
}
```

### ❌ Pattern 5: Repository Calling Service

```typescript
// BAD - Repository depends on service
@Injectable()
export class UserRepository {
  constructor(private readonly auditService: AuditService) {}

  async save(user: User): Promise<User> {
    const saved = await this.db.query('INSERT INTO users...', user);
    // VIOLATES ARCHITECTURE - Upward dependency
    await this.auditService.log('user_created', user.id);
    return saved;
  }
}

// ✅ GOOD - Service orchestrates repository + audit
@Injectable()
export class UserService {
  constructor(
    private readonly userRepository: UserRepository,
    private readonly auditService: AuditService,
  ) {}

  async create(dto: CreateUserDto): Promise<User> {
    const user = new User(dto);
    const saved = await this.userRepository.save(user);
    await this.auditService.log('user_created', saved.id);
    return saved;
  }
}
```

## Feature Module Isolation

### Module Boundaries

Each feature module should be isolated:

```typescript
// users.module.ts
@Module({
  controllers: [UsersController],
  providers: [UsersService, UsersRepository],
  exports: [UsersService], // Only export what's needed
})
export class UsersModule {}

// products.module.ts
@Module({
  controllers: [ProductsController],
  providers: [ProductsService, ProductsRepository],
  exports: [ProductsService],
})
export class ProductsModule {}
```

### Cross-Module Communication

```typescript
// ✅ GOOD - Through exported services
// orders.service.ts
@Injectable()
export class OrdersService {
  constructor(
    private readonly ordersRepository: OrdersRepository,
    private readonly usersService: UsersService, // From UsersModule.exports
    private readonly productsService: ProductsService, // From ProductsModule.exports
  ) {}

  async createOrder(dto: CreateOrderDto): Promise<Order> {
    // Use exported services only
    const user = await this.usersService.findById(dto.userId);
    const product = await this.productsService.findById(dto.productId);
    // ...
  }
}

// ❌ BAD - Accessing internal dependencies
@Injectable()
export class OrdersService {
  constructor(
    // VIOLATES BOUNDARIES - Accessing private repository
    private readonly usersRepository: UsersRepository,
  ) {}
}
```

## Validation Checklist

Use this when reviewing or generating code:

- [ ] Controllers only handle HTTP concerns
- [ ] Services contain all business logic
- [ ] Repositories only perform database operations
- [ ] No upward dependencies (repositories → services → controllers)
- [ ] No direct DB calls in controllers
- [ ] No HTTP concerns in services
- [ ] No business logic in repositories
- [ ] Entities are pure domain objects (no decorators beyond domain)
- [ ] Modules export only necessary dependencies
- [ ] No circular dependencies between modules
- [ ] Cross-module communication through exported services only
- [ ] Error handling appropriate to layer (HTTP exceptions in controllers)

## Generation Workflow

When generating a new feature, follow this order:

1. **Entity** (Domain layer)
   - Define pure domain model
   - Add domain methods only
   - No persistence concerns

2. **Repository** (Infrastructure layer)
   - Implement data access
   - Map between entity and persistence format
   - No business logic

3. **DTO** (Infrastructure layer)
   - Define input/output contracts
   - Add validation rules

4. **Service** (Application layer)
   - Implement business logic
   - Orchestrate repositories
   - Handle validation and transformations

5. **Controller** (Presentation layer)
   - Define endpoints
   - Delegate to service
   - Format HTTP responses

6. **Module** (Glue layer)
   - Wire dependencies
   - Export what's needed
   - Import required modules

## Code Review Workflow

When reviewing code for architectural violations:

1. **Check dependencies flow**: Can you trace imports downward only?
2. **Identify layer violations**: Is each class in the right layer?
3. **Verify isolation**: Are modules using only exported services?
4. **Look for leaks**: Any HTTP code in services? DB code in controllers?
5. **Check boundaries**: Are cross-module dependencies going through exports?
6. **Detect anti-patterns**: Circular deps? Upward dependencies?

## Additional Resources

- For naming and folder structure details, see [structure.md](structure.md)
- For violation examples and fixes, see [patterns.md](patterns.md)
- Pair with the `code-conventions` skill for tactical enforcement
- Reference `.cursor/rules/nestjs-standards.mdc` for architectural standards
