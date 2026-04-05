# Custom Error Classes

This document covers creating domain-specific error classes for common business scenarios.

## Error Hierarchy

```
Error
├── NestJS Built-in Exceptions
│   ├── BadRequestException (400)
│   ├── UnauthorizedException (401)
│   ├── ForbiddenException (403)
│   ├── NotFoundException (404)
│   ├── ConflictException (409)
│   └── InternalServerErrorException (500)
│
└── Custom AppError
    ├── ValidationError (400)
    ├── DuplicateError (409)
    ├── NotFoundError (404)
    ├── UnauthorizedError (401)
    ├── ForbiddenError (403)
    ├── InternalServerError (500)
    └── Domain-Specific Errors
        ├── InsufficientFundsError (402)
        ├── OutOfStockError (409)
        └── ...
```

## Base Error Classes

### HTTP Errors (NestJS Built-in)

```typescript
// src/common/errors/http.errors.ts
import {
  BadRequestException,
  UnauthorizedException,
  ForbiddenException,
  NotFoundException,
  ConflictException,
  HttpStatus,
} from '@nestjs/common';

export class ValidationError extends BadRequestException {
  constructor(message: string, details?: Record<string, any>) {
    super({
      statusCode: HttpStatus.BAD_REQUEST,
      error: 'ValidationError',
      message,
      ...(details && { details }),
    });
  }
}

export class InvalidInputError extends BadRequestException {
  constructor(field: string, reason: string) {
    super({
      statusCode: HttpStatus.BAD_REQUEST,
      error: 'InvalidInputError',
      message: `Invalid ${field}: ${reason}`,
      details: { field, reason },
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
  constructor(resource: string, id?: any) {
    const message = id
      ? `${resource} with id='${id}' not found`
      : `${resource} not found`;

    super({
      statusCode: HttpStatus.NOT_FOUND,
      error: 'NotFoundError',
      message,
      ...(id && { details: { resource, id } }),
    });
  }
}

export class UnauthorizedError extends UnauthorizedException {
  constructor(message: string = 'Authentication required', reason?: string) {
    super({
      statusCode: HttpStatus.UNAUTHORIZED,
      error: 'UnauthorizedError',
      message,
      ...(reason && { details: { reason } }),
    });
  }
}

export class ForbiddenError extends ForbiddenException {
  constructor(message: string = 'Access denied', reason?: string) {
    super({
      statusCode: HttpStatus.FORBIDDEN,
      error: 'ForbiddenError',
      message,
      ...(reason && { details: { reason } }),
    });
  }
}

export class TokenExpiredError extends UnauthorizedException {
  constructor() {
    super({
      statusCode: HttpStatus.UNAUTHORIZED,
      error: 'TokenExpiredError',
      message: 'Authentication token expired',
      details: { reason: 'token_expired' },
    });
  }
}

export class InvalidTokenError extends UnauthorizedException {
  constructor() {
    super({
      statusCode: HttpStatus.UNAUTHORIZED,
      error: 'InvalidTokenError',
      message: 'Invalid authentication token',
      details: { reason: 'invalid_token' },
    });
  }
}

export class InternalServerError extends Error {
  constructor(message: string = 'Internal server error', originalError?: Error) {
    super(message);
    this.name = 'InternalServerError';
    Object.setPrototypeOf(this, InternalServerError.prototype);
  }
}
```

## Domain-Specific Errors

### User Domain

```typescript
// src/users/errors/user.errors.ts
import { BadRequestException, HttpStatus } from '@nestjs/common';

export class EmailAlreadyInUseError extends ConflictException {
  constructor(email: string) {
    super({
      statusCode: HttpStatus.CONFLICT,
      error: 'EmailAlreadyInUseError',
      message: `Email '${email}' is already in use`,
      details: { email },
    });
  }
}

export class InvalidEmailError extends BadRequestException {
  constructor(email: string) {
    super({
      statusCode: HttpStatus.BAD_REQUEST,
      error: 'InvalidEmailError',
      message: `Invalid email address: ${email}`,
      details: { email },
    });
  }
}

export class InvalidPasswordError extends BadRequestException {
  constructor(reason: string) {
    super({
      statusCode: HttpStatus.BAD_REQUEST,
      error: 'InvalidPasswordError',
      message: `Password does not meet requirements: ${reason}`,
      details: { reason },
    });
  }
}

export class UserNotFoundError extends NotFoundException {
  constructor(userId: string) {
    super({
      statusCode: HttpStatus.NOT_FOUND,
      error: 'UserNotFoundError',
      message: `User '${userId}' not found`,
      details: { userId },
    });
  }
}

export class UserInactiveError extends ForbiddenException {
  constructor(userId: string) {
    super({
      statusCode: HttpStatus.FORBIDDEN,
      error: 'UserInactiveError',
      message: `User '${userId}' is inactive`,
      details: { userId },
    });
  }
}
```

### Product Domain

```typescript
// src/products/errors/product.errors.ts
import { BadRequestException, ConflictException, HttpStatus } from '@nestjs/common';

export class ProductNotFoundError extends NotFoundException {
  constructor(productId: string) {
    super({
      statusCode: HttpStatus.NOT_FOUND,
      error: 'ProductNotFoundError',
      message: `Product '${productId}' not found`,
      details: { productId },
    });
  }
}

export class OutOfStockError extends ConflictException {
  constructor(productId: string, requested: number, available: number) {
    super({
      statusCode: HttpStatus.CONFLICT,
      error: 'OutOfStockError',
      message: `Product '${productId}' out of stock (requested: ${requested}, available: ${available})`,
      details: { productId, requested, available },
    });
  }
}

export class InvalidPriceError extends BadRequestException {
  constructor(price: number, reason: string) {
    super({
      statusCode: HttpStatus.BAD_REQUEST,
      error: 'InvalidPriceError',
      message: `Invalid price ${price}: ${reason}`,
      details: { price, reason },
    });
  }
}
```

### Order Domain

```typescript
// src/orders/errors/order.errors.ts
import { BadRequestException, ConflictException, HttpStatus } from '@nestjs/common';

export class OrderNotFoundError extends NotFoundException {
  constructor(orderId: string) {
    super({
      statusCode: HttpStatus.NOT_FOUND,
      error: 'OrderNotFoundError',
      message: `Order '${orderId}' not found`,
      details: { orderId },
    });
  }
}

export class InvalidOrderStatusTransitionError extends ConflictException {
  constructor(from: string, to: string) {
    super({
      statusCode: HttpStatus.CONFLICT,
      error: 'InvalidOrderStatusTransitionError',
      message: `Cannot transition order from '${from}' to '${to}'`,
      details: { from, to },
    });
  }
}

export class InsufficientFundsError extends ConflictException {
  constructor(userId: string, required: number, available: number) {
    super({
      statusCode: HttpStatus.CONFLICT,
      error: 'InsufficientFundsError',
      message: `Insufficient funds (required: ${required}, available: ${available})`,
      details: { userId, required, available },
    });
  }
}

export class PaymentFailedError extends ConflictException {
  constructor(orderId: string, reason: string) {
    super({
      statusCode: HttpStatus.CONFLICT,
      error: 'PaymentFailedError',
      message: `Payment failed for order '${orderId}': ${reason}`,
      details: { orderId, reason },
    });
  }
}
```

### Auth Domain

```typescript
// src/auth/errors/auth.errors.ts
import { UnauthorizedException, ForbiddenException, HttpStatus, BadRequestException } from '@nestjs/common';

export class CredentialsInvalidError extends UnauthorizedException {
  constructor() {
    super({
      statusCode: HttpStatus.UNAUTHORIZED,
      error: 'CredentialsInvalidError',
      message: 'Invalid email or password',
      details: { reason: 'invalid_credentials' },
    });
  }
}

export class TooManyLoginAttemptsError extends ForbiddenException {
  constructor(lockoutMinutes: number) {
    super({
      statusCode: HttpStatus.FORBIDDEN,
      error: 'TooManyLoginAttemptsError',
      message: `Account locked due to too many login attempts. Try again in ${lockoutMinutes} minutes.`,
      details: { lockoutMinutes, reason: 'account_locked' },
    });
  }
}

export class SessionExpiredError extends UnauthorizedException {
  constructor() {
    super({
      statusCode: HttpStatus.UNAUTHORIZED,
      error: 'SessionExpiredError',
      message: 'Your session has expired. Please log in again.',
      details: { reason: 'session_expired' },
    });
  }
}

export class PermissionDeniedError extends ForbiddenException {
  constructor(permission: string) {
    super({
      statusCode: HttpStatus.FORBIDDEN,
      error: 'PermissionDeniedError',
      message: `You do not have permission: ${permission}`,
      details: { permission },
    });
  }
}

export class InvalidRefreshTokenError extends UnauthorizedException {
  constructor() {
    super({
      statusCode: HttpStatus.UNAUTHORIZED,
      error: 'InvalidRefreshTokenError',
      message: 'Invalid or expired refresh token',
      details: { reason: 'invalid_refresh_token' },
    });
  }
}
```

## External Service Errors

```typescript
// src/common/errors/external.errors.ts
import { BadRequestException, HttpStatus } from '@nestjs/common';

export class ExternalServiceError extends BadRequestException {
  constructor(service: string, message: string, statusCode?: number) {
    super({
      statusCode: statusCode || HttpStatus.BAD_GATEWAY,
      error: 'ExternalServiceError',
      message: `${service} error: ${message}`,
      details: { service },
    });
  }
}

export class PaymentGatewayError extends ExternalServiceError {
  constructor(message: string, statusCode?: number) {
    super('PaymentGateway', message, statusCode || 502);
  }
}

export class EmailServiceError extends ExternalServiceError {
  constructor(message: string) {
    super('EmailService', message, 503);
  }
}

export class ThirdPartyAPIError extends ExternalServiceError {
  constructor(apiName: string, message: string) {
    super(apiName, message, 502);
  }
}

export class ExternalServiceTimeoutError extends ExternalServiceError {
  constructor(service: string) {
    super(service, `Request timeout`, 504);
  }
}
```

## Database Errors

```typescript
// src/common/errors/database.errors.ts
import { BadRequestException, HttpStatus } from '@nestjs/common';

export class DatabaseError extends BadRequestException {
  constructor(message: string, originalError?: Error) {
    super({
      statusCode: HttpStatus.INTERNAL_SERVER_ERROR,
      error: 'DatabaseError',
      message: 'Database operation failed',
      details: {
        reason: message,
        ...(originalError && { originalErrorCode: (originalError as any).code }),
      },
    });
  }
}

export class UniqueConstraintError extends DatabaseError {
  constructor(field: string, value: any) {
    super(`Unique constraint violation on field: ${field}`);
    this.name = 'UniqueConstraintError';
  }
}

export class ForeignKeyConstraintError extends DatabaseError {
  constructor(resource: string, referencedResource: string) {
    super(`Cannot delete ${resource}: referenced by ${referencedResource}`);
    this.name = 'ForeignKeyConstraintError';
  }
}

export class DeadlockError extends DatabaseError {
  constructor() {
    super('Database deadlock - please retry');
    this.name = 'DeadlockError';
  }
}

export class ConnectionPoolExhaustedError extends DatabaseError {
  constructor() {
    super('Database connection pool exhausted');
    this.name = 'ConnectionPoolExhaustedError';
  }
}
```

## Validation Errors

```typescript
// src/common/errors/validation.errors.ts
import { BadRequestException, HttpStatus } from '@nestjs/common';

export class ValidationFailedError extends BadRequestException {
  constructor(
    public readonly violations: Record<string, string[]>,
  ) {
    super({
      statusCode: HttpStatus.BAD_REQUEST,
      error: 'ValidationFailedError',
      message: 'Validation failed',
      details: { violations },
    });
  }
}

export class FieldValidationError extends BadRequestException {
  constructor(field: string, constraint: string, value: any) {
    super({
      statusCode: HttpStatus.BAD_REQUEST,
      error: 'FieldValidationError',
      message: `Field '${field}' failed validation: ${constraint}`,
      details: { field, constraint, value },
    });
  }
}
```

## Error Factory Pattern

```typescript
// src/common/errors/error.factory.ts
import { HttpException, HttpStatus } from '@nestjs/common';

export class ErrorFactory {
  // HTTP Errors
  static badRequest(message: string, details?: any): HttpException {
    return new BadRequestException({
      statusCode: HttpStatus.BAD_REQUEST,
      error: 'BadRequest',
      message,
      ...(details && { details }),
    });
  }

  static unauthorized(message: string = 'Unauthorized'): HttpException {
    return new UnauthorizedError(message);
  }

  static forbidden(message: string = 'Forbidden'): HttpException {
    return new ForbiddenError(message);
  }

  static notFound(resource: string, id?: any): HttpException {
    return new NotFoundError(resource, id);
  }

  static conflict(message: string, details?: any): HttpException {
    return new ConflictException({
      statusCode: HttpStatus.CONFLICT,
      error: 'Conflict',
      message,
      ...(details && { details }),
    });
  }

  // Domain Errors
  static emailAlreadyInUse(email: string): HttpException {
    return new EmailAlreadyInUseError(email);
  }

  static outOfStock(productId: string, requested: number, available: number): HttpException {
    return new OutOfStockError(productId, requested, available);
  }

  static paymentFailed(orderId: string, reason: string): HttpException {
    return new PaymentFailedError(orderId, reason);
  }

  // External Service Errors
  static externalServiceError(service: string, message: string): HttpException {
    return new ExternalServiceError(service, message);
  }
}
```

## Usage in Services

```typescript
@Injectable()
export class UserService {
  async createUser(dto: CreateUserDto): Promise<User> {
    // Validate input
    if (!isValidEmail(dto.email)) {
      throw new InvalidEmailError(dto.email);
    }

    // Check duplicates
    const existing = await this.userRepository.findByEmail(dto.email);
    if (existing) {
      throw new EmailAlreadyInUseError(dto.email);
    }

    // Validate password
    const passwordError = this.validatePassword(dto.password);
    if (passwordError) {
      throw new InvalidPasswordError(passwordError);
    }

    // Create user
    try {
      const user = new User();
      user.email = dto.email;
      user.passwordHash = await this.hashPassword(dto.password);
      user.name = dto.name;

      return await this.userRepository.save(user);
    } catch (error) {
      if (error.code === 'UNIQUE_CONSTRAINT') {
        throw new EmailAlreadyInUseError(dto.email);
      }

      throw new DatabaseError('Failed to create user', error);
    }
  }

  async getUserById(id: string): Promise<User> {
    const user = await this.userRepository.findById(id);
    if (!user) {
      throw new UserNotFoundError(id);
    }
    return user;
  }
}
```
