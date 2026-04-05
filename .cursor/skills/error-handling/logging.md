# Logging Strategies & Configuration

This document covers comprehensive logging setup and best practices with NestJS-Pino.

## NestJS-Pino Setup

### Installation

```bash
npm install nestjs-pino pino pino-pretty pino-http
```

### Module Configuration

```typescript
// src/config/logger.config.ts
import { LoggerModuleAsyncParams } from 'nestjs-pino';
import { ConfigService } from '@nestjs/config';

export const loggerAsyncConfig: LoggerModuleAsyncParams = {
  useFactory: (configService: ConfigService) => ({
    pinoHttp: {
      level: configService.get('LOG_LEVEL') || 'info',
      transport: {
        target: 'pino-pretty',
        options: {
          colorize: true,
          translateTime: 'SYS:standard',
          ignore: 'pid,hostname',
          singleLine: process.env.NODE_ENV === 'production',
          messageFormat:
            process.env.NODE_ENV === 'production'
              ? undefined
              : '[{levelLabel}] {msg}',
        },
      },
      serializers: {
        req: (req) => ({
          id: req.id,
          method: req.method,
          url: req.url,
          query: req.query,
          headers: sanitizeHeaders(req.headers),
          body: sanitizeBody(req.body),
          userAgent: req.headers['user-agent'],
          ip: req.ip,
        }),
        res: (res) => ({
          statusCode: res.statusCode,
          responseTime: res.responseTime,
          headers: sanitizeHeaders(res.getHeaders?.()),
        }),
        error: pino.stdSerializers.err,
      },
    },
  }),
  inject: [ConfigService],
};

// Sanitization helpers
function sanitizeHeaders(headers: any): any {
  if (!headers) return {};

  const sensitive = [
    'authorization',
    'cookie',
    'x-api-key',
    'x-auth-token',
    'x-access-token',
  ];

  return Object.fromEntries(
    Object.entries(headers).filter(
      ([key]) => !sensitive.includes(key.toLowerCase())
    )
  );
}

function sanitizeBody(body: any): any {
  if (!body) return body;

  const sensitive = ['password', 'token', 'apiKey', 'secret', 'creditCard'];
  const sanitized = JSON.parse(JSON.stringify(body));

  function sanitizeObject(obj: any): any {
    for (const key in obj) {
      if (sensitive.some(s => key.toLowerCase().includes(s.toLowerCase()))) {
        obj[key] = '***REDACTED***';
      } else if (typeof obj[key] === 'object' && obj[key] !== null) {
        sanitizeObject(obj[key]);
      }
    }
    return obj;
  }

  return sanitizeObject(sanitized);
}
```

### App Module Setup

```typescript
// src/app.module.ts
import { Module } from '@nestjs/common';
import { LoggerModule } from 'nestjs-pino';
import { loggerAsyncConfig } from './config/logger.config';

@Module({
  imports: [
    LoggerModule.forRootAsync(loggerAsyncConfig),
    // ... other modules
  ],
})
export class AppModule {}
```

## Environment-Based Configuration

### .env.local (Development)

```bash
LOG_LEVEL=debug
NODE_ENV=development
LOG_FORMAT=pretty
DATABASE_LOGGING=true
```

### .env.staging

```bash
LOG_LEVEL=info
NODE_ENV=staging
LOG_FORMAT=json
DATABASE_LOGGING=false
```

### .env.production

```bash
LOG_LEVEL=warn
NODE_ENV=production
LOG_FORMAT=json
DATABASE_LOGGING=false
```

## Logging Patterns

### Debug Logging

```typescript
// For development only - disabled in production
@Injectable()
export class UserService {
  constructor(
    private readonly logger: Logger,
    private readonly userRepository: UserRepository,
  ) {}

  async createUser(dto: CreateUserDto): Promise<User> {
    // Debug: Function entry with parameters
    this.logger.debug('createUser called', { email: dto.email, age: dto.age });

    // Debug: Intermediate steps
    this.logger.debug('Validating email format', { email: dto.email });

    // Debug: Data lookups
    const existing = await this.userRepository.findByEmail(dto.email);
    this.logger.debug('Email check completed', { exists: !!existing });

    if (existing) {
      this.logger.debug('Duplicate user detected', { email: dto.email });
      throw new DuplicateError('User', 'email', dto.email);
    }

    const user = new User();
    user.email = dto.email;
    user.name = dto.name;

    const saved = await this.userRepository.save(user);

    // Debug: Return value
    this.logger.debug('User created', { userId: saved.id });

    return saved;
  }
}
```

### Info Logging (Business Events)

```typescript
@Injectable()
export class OrderService {
  async createOrder(dto: CreateOrderDto, userId: string): Promise<Order> {
    this.logger.log(
      `Order creation initiated for user: ${userId}`,
      'OrderService.createOrder'
    );

    // Business event
    const order = new Order();
    order.userId = userId;
    order.items = dto.items;

    const saved = await this.orderRepository.save(order);

    // Log successful completion
    this.logger.log(
      `Order created successfully: ${saved.id}`,
      'OrderService.createOrder'
    );

    // Event-level data
    this.logger.log('OrderCreatedEvent', {
      orderId: saved.id,
      userId,
      itemCount: dto.items.length,
      totalAmount: dto.totalAmount,
      timestamp: new Date().toISOString(),
    });

    return saved;
  }
}
```

### Warn Logging (Recoverable Issues)

```typescript
@Injectable()
export class AuthService {
  private loginAttempts = new Map<string, number>();

  async login(email: string, password: string): Promise<Token> {
    // Check login attempts
    const attempts = this.loginAttempts.get(email) || 0;

    if (attempts > 5) {
      this.logger.warn(
        `Excessive login attempts for email: ${email}`,
        { email, attempts, action: 'account_locked' }
      );

      throw new TooManyLoginAttemptsError(15);
    }

    // Verify credentials
    const user = await this.userRepository.findByEmail(email);

    if (!user || !(await this.verifyPassword(password, user.passwordHash))) {
      this.loginAttempts.set(email, attempts + 1);

      this.logger.warn(
        `Failed login attempt for email: ${email}`,
        { email, attempt: attempts + 1 }
      );

      throw new CredentialsInvalidError();
    }

    // Reset attempts on success
    this.loginAttempts.delete(email);

    const token = this.generateToken(user);

    this.logger.log(`User logged in: ${user.id}`);

    return token;
  }
}
```

### Error Logging

```typescript
@Injectable()
export class PaymentService {
  async processPayment(orderId: string, amount: number): Promise<Payment> {
    try {
      this.logger.debug(`Processing payment for order: ${orderId}`);

      const result = await this.stripeClient.charges.create({
        amount,
        currency: 'usd',
      });

      this.logger.log(
        `Payment processed successfully: ${orderId}`,
        { orderId, transactionId: result.id, amount }
      );

      return result;
    } catch (error) {
      // Log error with full context
      this.logger.error(
        `Payment processing failed for order: ${orderId}`,
        {
          orderId,
          amount,
          errorCode: error.code,
          errorMessage: error.message,
          stack: error.stack,
          retryable: error.retryable,
        }
      );

      // Handle specific error types
      if (error.code === 'card_declined') {
        throw new PaymentFailedError(orderId, 'Card declined');
      } else if (error.code === 'rate_limit') {
        this.logger.warn('Stripe rate limit hit', { retryAfter: error.headers?.['retry-after'] });
        throw new ExternalServiceError('Stripe', 'Rate limited', 429);
      }

      throw new PaymentFailedError(orderId, 'Payment processing failed');
    }
  }
}
```

## Structured Data Logging

### Context Information

```typescript
@Injectable()
export class UserService {
  constructor(
    private readonly logger: Logger,
    private readonly requestContext: RequestContextService,
  ) {}

  async getUser(id: string): Promise<User> {
    const traceId = this.requestContext.getTraceId();
    const userId = this.requestContext.getUserId();

    this.logger.debug(
      `Fetching user: ${id}`,
      {
        traceId,     // Request correlation
        requestedBy: userId,  // Who is requesting
        resource: 'user',
        resourceId: id,
        timestamp: new Date().toISOString(),
      }
    );

    const user = await this.userRepository.findById(id);

    if (!user) {
      this.logger.warn(
        `User not found: ${id}`,
        {
          traceId,
          resource: 'user',
          resourceId: id,
          status: 'not_found',
        }
      );

      throw new UserNotFoundError(id);
    }

    return user;
  }
}
```

### Performance Logging

```typescript
@Injectable()
export class OrderService {
  async createOrder(dto: CreateOrderDto): Promise<Order> {
    const startTime = Date.now();

    try {
      const order = new Order();
      // ... create order

      const saved = await this.orderRepository.save(order);

      const duration = Date.now() - startTime;

      this.logger.log(
        `Order created`,
        {
          orderId: saved.id,
          duration,
          slow: duration > 1000, // Flag slow operations
        }
      );

      return saved;
    } catch (error) {
      const duration = Date.now() - startTime;

      this.logger.error(
        `Order creation failed`,
        {
          duration,
          error: error.message,
          slow: duration > 1000,
        }
      );

      throw error;
    }
  }
}
```

## Log Levels by Layer

### Controller Layer

```typescript
@Controller('users')
export class UserController {
  constructor(
    private readonly userService: UserService,
    private readonly logger: Logger,
  ) {}

  @Post()
  async createUser(@Body() createUserDto: CreateUserDto) {
    // Log incoming request (debug)
    this.logger.debug(
      `POST /users - Creating user: ${createUserDto.email}`
    );

    try {
      const user = await this.userService.createUser(createUserDto);

      // Log successful response (info)
      this.logger.log(
        `User created successfully`,
        { userId: user.id, email: user.email }
      );

      return user;
    } catch (error) {
      // Error is logged by exception filter
      throw error;
    }
  }

  @Get(':id')
  async getUser(@Param('id') id: string) {
    this.logger.debug(`GET /users/:id - Retrieving user: ${id}`);
    return this.userService.getUser(id);
  }
}
```

### Service Layer

```typescript
@Injectable()
export class UserService {
  private readonly logger = new Logger(UserService.name);

  async createUser(dto: CreateUserDto): Promise<User> {
    // Debug: Method entry
    this.logger.debug(`Creating user: ${dto.email}`);

    try {
      // Info: Business logic
      this.logger.log(`Validating user data for: ${dto.email}`);

      // Debug: Intermediate steps
      const existing = await this.userRepository.findByEmail(dto.email);
      this.logger.debug(`Email check completed: ${!existing}`);

      if (existing) {
        // Warn: Known error case
        this.logger.warn(`Duplicate email attempt: ${dto.email}`);
        throw new DuplicateError('User', 'email', dto.email);
      }

      const user = new User();
      const saved = await this.userRepository.save(user);

      // Info: Success
      this.logger.log(`User created: ${saved.id}`);

      return saved;
    } catch (error) {
      // Error: Unexpected error
      if (!(error instanceof HttpException)) {
        this.logger.error(
          `Unexpected error creating user: ${dto.email}`,
          error.stack
        );
      }
      throw error;
    }
  }
}
```

### Repository Layer

```typescript
@Injectable()
export class UserRepository {
  private readonly logger = new Logger(UserRepository.name);

  async save(user: User): Promise<User> {
    this.logger.debug(`Saving user: ${user.email}`);

    try {
      const result = await this.db.query(
        'INSERT INTO users (email, name, age) VALUES (?, ?, ?)',
        [user.email, user.name, user.age]
      );

      this.logger.debug(`User saved with ID: ${result.id}`);
      return result;
    } catch (error) {
      this.logger.error(
        `Database error saving user: ${user.email}`,
        {
          errorCode: error.code,
          constraint: error.constraint,
          stack: error.stack,
        }
      );

      throw new DatabaseError('Failed to save user', error);
    }
  }
}
```

## Monitoring Log Output

### Development (Pretty-Printed)

```
[2026-04-05 11:30:45.123] INFO Creating user: user@test.com
[2026-04-05 11:30:45.124] DEBUG Email check completed: false
[2026-04-05 11:30:45.234] INFO User created: user_123
```

### Production (JSON)

```json
{"level":30,"time":1712324445123,"msg":"Creating user: user@test.com","service":"UserService","context":"createUser"}
{"level":30,"time":1712324445234,"msg":"User created","userId":"user_123","email":"user@test.com"}
```

## Log Aggregation Setup

### Example: Send to ELK Stack

```typescript
// src/config/logger.config.ts
import pino from 'pino';

const targets = [];

// Console output
targets.push({
  target: 'pino-pretty',
  options: { colorize: true, translateTime: 'SYS:standard' },
});

// ELK Stack
if (process.env.NODE_ENV === 'production') {
  targets.push({
    target: 'pino-elasticsearch',
    options: {
      node: process.env.ELASTICSEARCH_URL,
      auth: {
        username: process.env.ELASTICSEARCH_USER,
        password: process.env.ELASTICSEARCH_PASSWORD,
      },
    },
  });
}

export const loggerTransport = pino.transport({ targets });
```

## Best Practices Checklist

- [ ] Use appropriate log levels (debug, info, warn, error)
- [ ] Never log sensitive data (passwords, tokens, credit cards)
- [ ] Include request/trace IDs for correlation
- [ ] Log errors with full stack traces
- [ ] Include context data (user IDs, resource IDs, etc.)
- [ ] Log business events (create, update, delete)
- [ ] Log performance metrics (duration, slow operations)
- [ ] Sanitize request/response bodies
- [ ] Use structured logging (JSON in production)
- [ ] Monitor log output in production
- [ ] Set appropriate log levels per environment
- [ ] Don't log in loops (cause log explosion)
