---
name: security
description: Implement secure authentication with JWT, guards, and role-based access control. Enforce input validation with class-validator. Prevent common attacks (SQL injection, XSS, CSRF). Use bcrypt for passwords, HTTPS headers, and secure session management.
---

# Security

This skill guides implementing comprehensive security measures in NestJS projects, from authentication/authorization to input validation and attack prevention.

## Authentication with JWT

### JWT Configuration

```typescript
// src/config/jwt.config.ts
import { JwtModuleOptions } from '@nestjs/jwt';
import { ConfigService } from '@nestjs/config';

export const jwtConfig = (configService: ConfigService): JwtModuleOptions => ({
  secret: configService.get('JWT_SECRET'),
  signOptions: {
    expiresIn: configService.get('JWT_EXPIRATION') || '24h',
    algorithm: 'HS256',
    issuer: 'your-app',
  },
});

export const jwtRefreshConfig = (configService: ConfigService): JwtModuleOptions => ({
  secret: configService.get('JWT_REFRESH_SECRET'),
  signOptions: {
    expiresIn: configService.get('JWT_REFRESH_EXPIRATION') || '7d',
    algorithm: 'HS256',
  },
});
```

### JWT Service

```typescript
// src/auth/jwt.service.ts
import { Injectable } from '@nestjs/common';
import { JwtService } from '@nestjs/jwt';

@Injectable()
export class AuthTokenService {
  constructor(private readonly jwtService: JwtService) {}

  generateAccessToken(userId: string, email: string, roles: string[]): string {
    return this.jwtService.sign({
      sub: userId,
      email,
      roles,
      type: 'access',
    });
  }

  generateRefreshToken(userId: string): string {
    return this.jwtService.sign(
      { sub: userId, type: 'refresh' },
      { expiresIn: '7d', secret: process.env.JWT_REFRESH_SECRET }
    );
  }

  validateAccessToken(token: string): any {
    try {
      return this.jwtService.verify(token);
    } catch (error) {
      throw new InvalidTokenError();
    }
  }

  validateRefreshToken(token: string): any {
    try {
      return this.jwtService.verify(token, {
        secret: process.env.JWT_REFRESH_SECRET,
      });
    } catch (error) {
      throw new InvalidRefreshTokenError();
    }
  }
}
```

## Authentication Guards

### JWT Strategy

```typescript
// src/auth/strategies/jwt.strategy.ts
import { Injectable } from '@nestjs/common';
import { PassportStrategy } from '@nestjs/passport';
import { ExtractJwt, Strategy } from 'passport-jwt';
import { ConfigService } from '@nestjs/config';
import { UserService } from '../../users/user.service';

@Injectable()
export class JwtStrategy extends PassportStrategy(Strategy) {
  constructor(
    configService: ConfigService,
    private readonly userService: UserService,
  ) {
    super({
      jwtFromRequest: ExtractJwt.fromAuthHeaderAsBearerToken(),
      ignoreExpiration: false,
      secretOrKey: configService.get('JWT_SECRET'),
    });
  }

  async validate(payload: any) {
    const user = await this.userService.findById(payload.sub);
    if (!user) {
      throw new UnauthorizedException('User not found');
    }
    return {
      id: payload.sub,
      email: payload.email,
      roles: payload.roles,
      user,
    };
  }
}
```

### JWT Auth Guard

```typescript
// src/auth/guards/jwt.guard.ts
import { Injectable } from '@nestjs/common';
import { AuthGuard } from '@nestjs/passport';

@Injectable()
export class JwtAuthGuard extends AuthGuard('jwt') {}
```

### Optional JWT Guard (for public endpoints with optional auth)

```typescript
// src/auth/guards/optional-jwt.guard.ts
import { Injectable } from '@nestjs/common';
import { AuthGuard } from '@nestjs/passport';

@Injectable()
export class OptionalJwtAuthGuard extends AuthGuard('jwt') {
  handleRequest(err, user) {
    return user || null; // Allow null users
  }
}
```

### Role-Based Access Control (RBAC) Guard

```typescript
// src/auth/guards/roles.guard.ts
import { Injectable, CanActivate, ExecutionContext, ForbiddenException } from '@nestjs/common';
import { Reflector } from '@nestjs/core';

@Injectable()
export class RolesGuard implements CanActivate {
  constructor(private reflector: Reflector) {}

  canActivate(context: ExecutionContext): boolean {
    const requiredRoles = this.reflector.get<string[]>(
      'roles',
      context.getHandler(),
    );

    if (!requiredRoles) return true; // No roles required

    const request = context.switchToHttp().getRequest();
    const user = request.user;

    if (!user || !user.roles) {
      throw new ForbiddenException('No roles assigned');
    }

    const hasRole = requiredRoles.some(role => user.roles.includes(role));
    if (!hasRole) {
      throw new ForbiddenException(`Required roles: ${requiredRoles.join(', ')}`);
    }

    return true;
  }
}
```

### Roles Decorator

```typescript
// src/auth/decorators/roles.decorator.ts
import { SetMetadata } from '@nestjs/common';

export const Roles = (...roles: string[]) => SetMetadata('roles', roles);
```

### Usage Example

```typescript
@Controller('users')
export class UserController {
  constructor(private readonly userService: UserService) {}

  @Get()
  @UseGuards(JwtAuthGuard, RolesGuard)
  @Roles('admin', 'moderator')
  async getAllUsers() {
    return this.userService.findAll();
  }

  @Post()
  @UseGuards(JwtAuthGuard)
  async createUser(@Body() dto: CreateUserDto, @Request() req) {
    return this.userService.createUser(dto, req.user.id);
  }

  @Get('/profile')
  @UseGuards(JwtAuthGuard)
  async getProfile(@Request() req) {
    return this.userService.findById(req.user.id);
  }
}
```

## Password Security with Bcrypt

### Hash Service

```typescript
// src/auth/password.service.ts
import { Injectable } from '@nestjs/common';
import * as bcrypt from 'bcrypt';

@Injectable()
export class PasswordService {
  private readonly saltRounds = 12; // Adjust based on security/performance needs

  async hash(password: string): Promise<string> {
    return bcrypt.hash(password, this.saltRounds);
  }

  async verify(password: string, hash: string): Promise<boolean> {
    return bcrypt.compare(password, hash);
  }

  validateStrength(password: string): { valid: boolean; errors: string[] } {
    const errors: string[] = [];

    if (password.length < 12) {
      errors.push('Password must be at least 12 characters');
    }

    if (!/[A-Z]/.test(password)) {
      errors.push('Password must contain uppercase letter');
    }

    if (!/[a-z]/.test(password)) {
      errors.push('Password must contain lowercase letter');
    }

    if (!/[0-9]/.test(password)) {
      errors.push('Password must contain number');
    }

    if (!/[!@#$%^&*]/.test(password)) {
      errors.push('Password must contain special character (!@#$%^&*)');
    }

    return {
      valid: errors.length === 0,
      errors,
    };
  }
}
```

## Input Validation

### DTOs with Validation

```typescript
// src/users/dto/create-user.dto.ts
import {
  IsEmail,
  IsString,
  MinLength,
  MaxLength,
  Matches,
  IsInt,
  Min,
  Max,
} from 'class-validator';

export class CreateUserDto {
  @IsEmail({}, { message: 'Invalid email format' })
  email: string;

  @IsString()
  @MinLength(2)
  @MaxLength(100)
  name: string;

  @IsString()
  @MinLength(12, { message: 'Password must be at least 12 characters' })
  @Matches(/[A-Z]/, { message: 'Password must contain uppercase letter' })
  @Matches(/[a-z]/, { message: 'Password must contain lowercase letter' })
  @Matches(/[0-9]/, { message: 'Password must contain number' })
  @Matches(/[!@#$%^&*]/, { message: 'Password must contain special character' })
  password: string;

  @IsInt()
  @Min(0)
  @Max(150)
  age: number;
}
```

### Global Validation Pipe

```typescript
// src/main.ts
import { ValidationPipe } from '@nestjs/common';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);

  app.useGlobalPipes(
    new ValidationPipe({
      whitelist: true,           // Strip unknown properties
      forbidNonWhitelisted: true, // Reject unknown properties
      transform: true,           // Auto-transform payloads
      transformOptions: {
        enableImplicitConversion: true,
      },
    })
  );

  await app.listen(3000);
}

bootstrap();
```

## CSRF Protection

### CSRF Token Guard

```typescript
// src/auth/guards/csrf.guard.ts
import { Injectable, CanActivate, ExecutionContext, BadRequestException } from '@nestjs/common';
import { Request } from 'express';

@Injectable()
export class CsrfGuard implements CanActivate {
  canActivate(context: ExecutionContext): boolean {
    const request = context.switchToHttp().getRequest<Request>();

    // Skip for GET requests
    if (request.method === 'GET') return true;

    const token = request.headers['x-csrf-token'];
    const sessionToken = request.session?.csrfToken;

    if (!token || !sessionToken || token !== sessionToken) {
      throw new BadRequestException('Invalid CSRF token');
    }

    return true;
  }
}
```

### CSRF Middleware

```typescript
// src/common/middleware/csrf.middleware.ts
import { Injectable, NestMiddleware } from '@nestjs/common';
import { Request, Response, NextFunction } from 'express';
import { v4 as uuidv4 } from 'uuid';

@Injectable()
export class CsrfMiddleware implements NestMiddleware {
  use(req: Request, res: Response, next: NextFunction) {
    if (!req.session) {
      req.session = {};
    }

    // Generate CSRF token if not exists
    if (!req.session.csrfToken) {
      req.session.csrfToken = uuidv4();
    }

    // Add token to response header
    res.setHeader('X-CSRF-Token', req.session.csrfToken);

    next();
  }
}
```

## Security Headers

### Helmet Configuration

```typescript
// src/main.ts
import helmet from 'helmet';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);

  // Apply security headers
  app.use(helmet({
    contentSecurityPolicy: {
      directives: {
        defaultSrc: ["'self'"],
        scriptSrc: ["'self'"],
        styleSrc: ["'self'", "'unsafe-inline'"],
        imgSrc: ["'self'", 'data:', 'https:'],
      },
    },
    hsts: {
      maxAge: 31536000, // 1 year
      includeSubDomains: true,
      preload: true,
    },
    frameguard: { action: 'deny' },
    noSniff: true,
    xssFilter: true,
  }));

  await app.listen(3000);
}
```

## Common Attack Prevention

### SQL Injection Prevention

```typescript
// ✅ GOOD - Using parameterized queries
const user = await this.db.query(
  'SELECT * FROM users WHERE email = ? AND id = ?',
  [email, id]
);

// ❌ BAD - String concatenation (vulnerable!)
const user = await this.db.query(
  `SELECT * FROM users WHERE email = '${email}' AND id = ${id}`
);
```

### XSS Prevention

```typescript
// ✅ GOOD - Framework handles escaping
const template = `<div>${user.name}</div>`; // Automatically escaped

// ✅ GOOD - Sanitize HTML if needed
import DOMPurify from 'dompurify';
const clean = DOMPurify.sanitize(userHtml);

// ❌ BAD - Trusting unsanitized input
res.json({ html: userInput }); // Could contain XSS
```

### Rate Limiting

```typescript
// src/main.ts
import rateLimit from 'express-rate-limit';

const limiter = rateLimit({
  windowMs: 15 * 60 * 1000, // 15 minutes
  max: 100,                 // Limit per IP
  message: 'Too many requests',
});

app.use('/api/', limiter);

// Stricter limit for auth endpoints
const authLimiter = rateLimit({
  windowMs: 15 * 60 * 1000,
  max: 5,
  message: 'Too many login attempts',
});

app.post('/auth/login', authLimiter, ...);
```

## Generation Checklist

When implementing security features:

- [ ] JWT configured with appropriate expiration times
- [ ] Refresh tokens stored securely (httpOnly cookies)
- [ ] Passwords hashed with bcrypt (salt rounds ≥ 10)
- [ ] Password strength validated before storage
- [ ] All input validated with DTOs and class-validator
- [ ] Guard applied to protected endpoints
- [ ] Role-based access control implemented
- [ ] CSRF token generated and validated
- [ ] Security headers applied (Helmet)
- [ ] Rate limiting configured
- [ ] HTTPS enforced in production
- [ ] Secrets in environment variables (never hardcoded)
- [ ] Sensitive data not logged

## Review Checklist

When reviewing security code:

- [ ] All private endpoints have `@UseGuards(JwtAuthGuard)`
- [ ] Roles are properly checked with `RolesGuard`
- [ ] Passwords are hashed, never stored plain text
- [ ] DTOs validate all input (no `any` types)
- [ ] Whitelist mode enabled on `ValidationPipe`
- [ ] CSRF tokens validated for state-changing requests
- [ ] No secrets in code or logs
- [ ] Error messages don't leak information
- [ ] SQL queries use parameterized queries
- [ ] HTTPS configured in production
- [ ] Rate limiting on sensitive endpoints
- [ ] No direct access to database queries from controllers

## Additional Resources

- For attack vector details and prevention, see [attacks.md](attacks.md)
- For authentication workflows, see [authentication.md](authentication.md)
- For authorization patterns, see [authorization.md](authorization.md)
