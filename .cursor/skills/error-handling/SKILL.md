---
name: error-handling
description: Implement global exception filtering, structured logging with NestJS-Pino, and consistent error response formats. Enforce custom error classes, proper error logging, and standardized API error contracts.
---

# Error Handling & Logging

This skill guides implementing comprehensive error handling and structured logging in NestJS projects with consistent response formats and logging levels.

## Standard Error Response Format

All API errors must return this format:

```json
{
  "statusCode": 400,
  "error": "BadRequestException",
  "message": "Email is required",
  "timestamp": "2026-04-05T11:30:00.000Z",
  "path": "/users",
  "traceId": "req_123abc",
  "details": {
    "field": "email",
    "constraint": "isEmail"
  }
}
```

## Global Exception Filter

### Implementation

```typescript
// src/common/filters/http-exception.filter.ts
import {
  ExceptionFilter,
  Catch,
  ArgumentsHost,
  HttpException,
  HttpStatus,
  Logger,
  BadRequestException,
  UnauthorizedException,
  ForbiddenException,
  NotFoundException,
  ConflictException,
} from '@nestjs/common';
import { Request, Response } from 'express';
import { v4 as uuidv4 } from 'uuid';

@Catch(HttpException)
export class HttpExceptionFilter implements ExceptionFilter {
  private readonly logger = new Logger(HttpExceptionFilter.name);

  catch(exception: HttpException, host: ArgumentsHost) {
    const ctx = host.switchToHttp();
    const response = ctx.getResponse<Response>();
    const request = ctx.getRequest<Request>();
    const status = exception.getStatus();
    const traceId = uuidv4();

    // Extract error details
    const exceptionResponse = exception.getResponse();
    const errorMessage =
      typeof exceptionResponse === 'object' && 'message' in exceptionResponse
        ? (exceptionResponse as any).message
        : exception.message;

    const errorDetails =
      typeof exceptionResponse === 'object' && 'error' in exceptionResponse
        ? (exceptionResponse as any)
        : {};

    // Build response
    const errorResponse = {
      statusCode: status,
      error: exception.name || 'HttpException',
      message: Array.isArray(errorMessage) ? errorMessage[0] : errorMessage,
      timestamp: new Date().toISOString(),
      path: request.url,
      traceId,
      ...(Object.keys(errorDetails).length > 0 && { details: errorDetails }),
    };

    // Log error
    this.logError(exception, request, errorResponse);

    // Send response
    response.status(status).json(errorResponse);
  }

  private logError(
    exception: HttpException,
    request: Request,
    errorResponse: any,
  ): void {
    const level = this.getLogLevel(exception.getStatus());

    this.logger[level](
      {
        traceId: errorResponse.traceId,
        method: request.method,
        path: request.url,
        statusCode: exception.getStatus(),
        error: exception.name,
        message: exception.message,
        stack: exception.stack,
      },
      `${request.method} ${request.url} - ${exception.getStatus()}`,
    );
  }

  private getLogLevel(statusCode: number): 'warn' | 'error' {
    return statusCode >= 500 ? 'error' : 'warn';
  }
}

// Register in app.module.ts
import { APP_FILTER } from '@nestjs/core';

@Module({
  providers: [
    {
      provide: APP_FILTER,
      useClass: HttpExceptionFilter,
    },
  ],
})
export class AppModule {}
```

## Custom Error Classes

### Base Error Class

```typescript
// src/common/errors/app.error.ts
export abstract class AppError extends Error {
  abstract readonly statusCode: number;
  abstract readonly isOperational: boolean;

  constructor(message: string) {
    super(message);
    Object.setPrototypeOf(this, AppError.prototype);
  }
}
```

### Domain-Specific Errors

```typescript
// src/common/errors/index.ts
import { BadRequestException, HttpStatus } from '@nestjs/common';

export class ValidationError extends BadRequestException {
  constructor(message: string, public readonly details?: Record<string, any>) {
    super({
      statusCode: HttpStatus.BAD_REQUEST,
      error: 'ValidationError',
      message,
      ...(details && { details }),
    });
  }
}

export class DuplicateError extends ConflictException {
  constructor(resource: string, field: string, value: any) {
    super({
      statusCode: HttpStatus.CONFLICT,
      error: 'DuplicateError',
      message: `${resource} with ${field}='${value}' already exists`,
      details: { resource, field, value },
    });
  }
}

export class NotFoundError extends NotFoundException {
  constructor(resource: string, id: any) {
    super({
      statusCode: HttpStatus.NOT_FOUND,
      error: 'NotFoundError',
      message: `${resource} with id='${id}' not found`,
      details: { resource, id },
    });
  }
}

export class UnauthorizedError extends UnauthorizedException {
  constructor(message: string = 'Authentication required') {
    super({
      statusCode: HttpStatus.UNAUTHORIZED,
      error: 'UnauthorizedError',
      message,
    });
  }
}

export class ForbiddenError extends ForbiddenException {
  constructor(message: string = 'Access denied') {
    super({
      statusCode: HttpStatus.FORBIDDEN,
      error: 'ForbiddenError',
      message,
    });
  }
}

export class InternalServerError extends BadRequestException {
  constructor(message: string = 'Internal server error', originalError?: Error) {
    super({
      statusCode: HttpStatus.INTERNAL_SERVER_ERROR,
      error: 'InternalServerError',
      message,
      ...(originalError && { originalError: originalError.message }),
    });
  }
}
```

## Structured Logging with NestJS-Pino

### Setup

```bash
npm install nestjs-pino pino pino-pretty
```

### Configuration

```typescript
// src/config/logger.config.ts
import { LogLevel } from '@nestjs/common';

export const loggerConfig = {
  pinoHttp: {
    level: process.env.LOG_LEVEL || 'info',
    transport: {
      target: 'pino-pretty',
      options: {
        colorize: true,
        translateTime: 'SYS:standard',
        ignore: 'pid,hostname',
        singleLine: false,
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
      }),
      res: (res) => ({
        statusCode: res.statusCode,
        responseTime: res.responseTime,
      }),
    },
  },
};

function sanitizeHeaders(headers: any) {
  const sensitive = ['authorization', 'cookie', 'x-api-key'];
  return Object.fromEntries(
    Object.entries(headers).filter(([key]) => !sensitive.includes(key.toLowerCase()))
  );
}

function sanitizeBody(body: any) {
  if (!body) return body;
  const sanitized = { ...body };
  const sensitive = ['password', 'token', 'apiKey', 'secret'];
  sensitive.forEach(key => {
    if (key in sanitized) {
      sanitized[key] = '***REDACTED***';
    }
  });
  return sanitized;
}
```

### Module Setup

```typescript
// src/app.module.ts
import { Module } from '@nestjs/common';
import { LoggerModule } from 'nestjs-pino';
import { loggerConfig } from './config/logger.config';

@Module({
  imports: [
    LoggerModule.forRoot(loggerConfig),
    // ... other modules
  ],
})
export class AppModule {}
```

## Logging Patterns

### Service Logging

```typescript
// src/users/user.service.ts
import { Logger } from '@nestjs/common';

@Injectable()
export class UserService {
  private readonly logger = new Logger(UserService.name);

  async createUser(dto: CreateUserDto): Promise<User> {
    this.logger.debug(`Creating user with email: ${dto.email}`);

    try {
      const existing = await this.userRepository.findByEmail(dto.email);
      if (existing) {
        this.logger.warn(`Duplicate user creation attempt: ${dto.email}`);
        throw new DuplicateError('User', 'email', dto.email);
      }

      const user = new User();
      user.email = dto.email;
      user.name = dto.name;
      user.age = dto.age;

      const saved = await this.userRepository.save(user);

      this.logger.log(`User created successfully: ${saved.id} (${saved.email})`);

      return saved;
    } catch (error) {
      this.logger.error(
        `Failed to create user: ${dto.email}`,
        error.stack,
      );
      throw error;
    }
  }

  async getUserById(id: string): Promise<User> {
    this.logger.debug(`Fetching user: ${id}`);

    const user = await this.userRepository.findById(id);
    if (!user) {
      this.logger.warn(`User not found: ${id}`);
      throw new NotFoundError('User', id);
    }

    this.logger.debug(`User retrieved: ${id}`);
    return user;
  }
}
```

### Controller Logging

```typescript
// src/users/user.controller.ts
import { Logger } from '@nestjs/common';

@Controller('users')
export class UserController {
  private readonly logger = new Logger(UserController.name);

  constructor(private readonly userService: UserService) {}

  @Post()
  async createUser(@Body() createUserDto: CreateUserDto) {
    this.logger.log(`Creating user: ${createUserDto.email}`);
    return this.userService.createUser(createUserDto);
  }

  @Get(':id')
  async getUser(@Param('id') id: string) {
    this.logger.debug(`Retrieving user: ${id}`);
    return this.userService.getUserById(id);
  }
}
```

## Logging Levels

| Level | Usage | Example |
|-------|-------|---------|
| **debug** | Development info, function entry | `Entering method X`, `Processing data Y` |
| **info** | Business events | `User created`, `Order processed` |
| **warn** | Recoverable issues | `Duplicate request`, `Retry attempt 3` |
| **error** | Errors needing attention | `Database error`, `External service failure` |

### Level Mapping

```typescript
export const logLevelMap = {
  debug: 0,   // Verbose, development only
  info: 1,    // Important business events
  warn: 2,    // Warnings (client errors 4xx)
  error: 3,   // Errors (server errors 5xx)
};

// Environment-based
LOG_LEVEL=debug   # Development
LOG_LEVEL=info    # Production
```

## Error Handling in Repositories

```typescript
// src/users/user.repository.ts
@Injectable()
export class UserRepository {
  private readonly logger = new Logger(UserRepository.name);

  constructor(private readonly db: Database) {}

  async save(user: User): Promise<User> {
    try {
      this.logger.debug(`Saving user: ${user.email}`);
      return await this.db.query('INSERT INTO users...', user);
    } catch (error) {
      if (error.code === 'DUPLICATE_KEY') {
        this.logger.warn(`Duplicate user email: ${user.email}`);
        throw new DuplicateError('User', 'email', user.email);
      }

      this.logger.error(`Database save failed: ${error.message}`, error.stack);
      throw new InternalServerError('Failed to save user', error);
    }
  }

  async findById(id: string): Promise<User | null> {
    try {
      return await this.db.query('SELECT * FROM users WHERE id = ?', [id]);
    } catch (error) {
      this.logger.error(`Database query failed: ${error.message}`, error.stack);
      throw new InternalServerError('Failed to query database', error);
    }
  }
}
```

## Error Handling in Services

```typescript
// Best practices
@Injectable()
export class OrderService {
  private readonly logger = new Logger(OrderService.name);

  async createOrder(dto: CreateOrderDto, userId: string): Promise<Order> {
    const startTime = Date.now();

    try {
      // Validate input
      const user = await this.userService.getUser(userId);
      if (!user) {
        throw new NotFoundError('User', userId);
      }

      // Business logic
      const order = new Order();
      order.userId = userId;
      order.items = dto.items;

      // Persist
      const saved = await this.orderRepository.save(order);

      this.logger.log(
        `Order created: ${saved.id} for user ${userId} in ${Date.now() - startTime}ms`
      );

      return saved;
    } catch (error) {
      // Re-throw known errors
      if (error instanceof AppError) {
        this.logger.warn(
          `Known error in createOrder: ${error.message}`
        );
        throw error;
      }

      // Log unexpected errors
      this.logger.error(
        `Unexpected error in createOrder for user ${userId}`,
        error.stack
      );

      throw new InternalServerError('Failed to create order', error);
    }
  }
}
```

## Centralized Error Handling

### Error Interceptor

```typescript
// src/common/interceptors/error.interceptor.ts
import { Injectable, NestInterceptor, ExecutionContext, CallHandler } from '@nestjs/common';
import { Observable, throwError } from 'rxjs';
import { catchError } from 'rxjs/operators';

@Injectable()
export class ErrorInterceptor implements NestInterceptor {
  intercept(_context: ExecutionContext, next: CallHandler): Observable<any> {
    return next.handle().pipe(
      catchError(error => {
        // Handle unknown errors
        if (!(error instanceof HttpException)) {
          return throwError(
            () => new InternalServerError('An unexpected error occurred', error)
          );
        }
        return throwError(() => error);
      })
    );
  }
}

// Register in app.module.ts
@Module({
  providers: [
    {
      provide: APP_INTERCEPTOR,
      useClass: ErrorInterceptor,
    },
  ],
})
export class AppModule {}
```

## Structured Error Context

### Request-Scoped Context

```typescript
// src/common/context/request.context.ts
import { Injectable } from '@nestjs/common';
import { AsyncLocalStorage } from 'async_hooks';

interface RequestContext {
  traceId: string;
  userId?: string;
  correlationId?: string;
}

@Injectable()
export class RequestContextService {
  private readonly asyncLocalStorage = new AsyncLocalStorage<RequestContext>();

  run<T>(context: RequestContext, callback: () => T): T {
    return this.asyncLocalStorage.run(context, callback);
  }

  getContext(): RequestContext {
    return this.asyncLocalStorage.getStore() || { traceId: 'unknown' };
  }

  getTraceId(): string {
    return this.getContext().traceId;
  }

  getUserId(): string | undefined {
    return this.getContext().userId;
  }
}

// Use in services
@Injectable()
export class UserService {
  constructor(
    private readonly requestContext: RequestContextService,
    private readonly logger: Logger,
  ) {}

  async getUser(id: string) {
    const traceId = this.requestContext.getTraceId();
    this.logger.debug(`Fetching user ${id}`, { traceId });
    // ...
  }
}
```

## Generation Checklist

When implementing error handling in a new service/controller:

- [ ] Use custom error classes (ValidationError, NotFoundError, etc.)
- [ ] Catch all errors and log appropriately
- [ ] Re-throw or transform to AppError
- [ ] Include context in logs (userId, resource id, etc.)
- [ ] Never expose sensitive data in error messages
- [ ] Use appropriate log levels (debug, info, warn, error)
- [ ] Log stack traces for unexpected errors
- [ ] Include timing information for performance tracking
- [ ] Add structured data (IDs, counts, etc.) for debugging
- [ ] Test both success and error paths

## Review Checklist

When reviewing error handling code:

- [ ] All errors caught and logged
- [ ] Sensitive data not logged (passwords, tokens, etc.)
- [ ] Consistent error response format
- [ ] Appropriate HTTP status codes
- [ ] Meaningful error messages
- [ ] Stack traces logged for server errors
- [ ] No silent failures (catch without logging)
- [ ] Custom errors used where appropriate
- [ ] Log levels used correctly
- [ ] Context data included for debugging

## Additional Resources

- For custom error class patterns, see [custom-errors.md](custom-errors.md)
- For logging strategies and configuration, see [logging.md](logging.md)
- For monitoring and alerting setup, see [monitoring.md](monitoring.md)
