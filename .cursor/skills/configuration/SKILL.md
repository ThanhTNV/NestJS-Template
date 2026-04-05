---
name: configuration
description: Centralize NestJS application configuration using @nestjs/config. Enforce .env files for secrets, environment-specific configs for dev/staging/production, and type-safe configuration access.
---

# Configuration & Environment

This skill guides implementing centralized, type-safe configuration management in NestJS projects with environment-specific settings and secret management.

## Directory Structure

```
src/config/
├── config.interface.ts    # Type definitions
├── app.config.ts          # App configuration
├── database.config.ts     # Database configuration
├── jwt.config.ts          # JWT configuration
├── cache.config.ts        # Cache configuration
└── index.ts               # Barrel export
```

## Setup

### Installation

```bash
npm install @nestjs/config joi
```

### Environment Files

```
.env                # Loaded: never (git-ignored)
.env.local          # Loaded: development (git-ignored)
.env.staging        # Loaded: staging (git-ignored)
.env.production     # Loaded: production (git-ignored)
.env.example        # Git-tracked template
```

### .env.example (Git-tracked template)

```bash
# App
NODE_ENV=development
APP_NAME=MyApp
APP_PORT=3000
APP_URL=http://localhost:3000

# Database
DATABASE_HOST=localhost
DATABASE_PORT=5432
DATABASE_USERNAME=user
DATABASE_PASSWORD=password
DATABASE_NAME=myapp_dev

# JWT
JWT_SECRET=your-secret-key-change-in-production
JWT_EXPIRATION=24h
JWT_REFRESH_SECRET=your-refresh-secret
JWT_REFRESH_EXPIRATION=7d

# Cache
CACHE_TTL=3600

# Logging
LOG_LEVEL=debug
```

## Configuration Interface

### Type Definitions

```typescript
// src/config/config.interface.ts
export interface IConfig {
  app: IAppConfig;
  database: IDatabaseConfig;
  jwt: IJwtConfig;
  cache: ICacheConfig;
  logging: ILoggingConfig;
}

export interface IAppConfig {
  name: string;
  port: number;
  env: 'development' | 'staging' | 'production';
  url: string;
}

export interface IDatabaseConfig {
  host: string;
  port: number;
  username: string;
  password: string;
  database: string;
  synchronize: boolean;
}

export interface IJwtConfig {
  secret: string;
  expiration: string;
  refreshSecret: string;
  refreshExpiration: string;
}

export interface ICacheConfig {
  ttl: number;
}

export interface ILoggingConfig {
  level: string;
}
```

## Configuration Modules

### App Configuration

```typescript
// src/config/app.config.ts
import { registerAs } from '@nestjs/config';
import { IAppConfig } from './config.interface';

export default registerAs('app', (): IAppConfig => ({
  name: process.env.APP_NAME || 'MyApp',
  port: parseInt(process.env.APP_PORT || '3000'),
  env: (process.env.NODE_ENV as 'development' | 'staging' | 'production') || 'development',
  url: process.env.APP_URL || 'http://localhost:3000',
}));
```

### Database Configuration

```typescript
// src/config/database.config.ts
import { registerAs } from '@nestjs/config';
import { IDatabaseConfig } from './config.interface';

export default registerAs('database', (): IDatabaseConfig => ({
  host: process.env.DATABASE_HOST || 'localhost',
  port: parseInt(process.env.DATABASE_PORT || '5432'),
  username: process.env.DATABASE_USERNAME || 'user',
  password: process.env.DATABASE_PASSWORD || 'password',
  database: process.env.DATABASE_NAME || 'myapp_dev',
  synchronize: process.env.NODE_ENV === 'development',
}));
```

### JWT Configuration

```typescript
// src/config/jwt.config.ts
import { registerAs } from '@nestjs/config';
import { IJwtConfig } from './config.interface';

export default registerAs('jwt', (): IJwtConfig => ({
  secret: process.env.JWT_SECRET || 'dev-secret-change-me',
  expiration: process.env.JWT_EXPIRATION || '24h',
  refreshSecret: process.env.JWT_REFRESH_SECRET || 'dev-refresh-secret',
  refreshExpiration: process.env.JWT_REFRESH_EXPIRATION || '7d',
}));
```

### Barrel Export

```typescript
// src/config/index.ts
export { default as appConfig } from './app.config';
export { default as databaseConfig } from './database.config';
export { default as jwtConfig } from './jwt.config';
export { default as cacheConfig } from './cache.config';
export * from './config.interface';
```

## ConfigModule Setup

### Module Registration

```typescript
// src/app.module.ts
import { Module } from '@nestjs/common';
import { ConfigModule } from '@nestjs/config';
import * as Joi from 'joi';
import {
  appConfig,
  databaseConfig,
  jwtConfig,
  cacheConfig,
} from './config';

@Module({
  imports: [
    ConfigModule.forRoot({
      isGlobal: true,
      envFilePath: [
        `.env.${process.env.NODE_ENV}.local`,
        `.env.${process.env.NODE_ENV}`,
        '.env.local',
        '.env',
      ],
      load: [appConfig, databaseConfig, jwtConfig, cacheConfig],
      validationSchema: Joi.object({
        // App
        NODE_ENV: Joi.string()
          .valid('development', 'staging', 'production')
          .required(),
        APP_NAME: Joi.string().required(),
        APP_PORT: Joi.number().default(3000),
        APP_URL: Joi.string().required(),

        // Database
        DATABASE_HOST: Joi.string().required(),
        DATABASE_PORT: Joi.number().default(5432),
        DATABASE_USERNAME: Joi.string().required(),
        DATABASE_PASSWORD: Joi.string().required(),
        DATABASE_NAME: Joi.string().required(),

        // JWT
        JWT_SECRET: Joi.string().required(),
        JWT_EXPIRATION: Joi.string().default('24h'),
        JWT_REFRESH_SECRET: Joi.string().required(),
        JWT_REFRESH_EXPIRATION: Joi.string().default('7d'),

        // Logging
        LOG_LEVEL: Joi.string()
          .valid('error', 'warn', 'info', 'debug', 'verbose')
          .default('info'),
      }),
      validationOptions: {
        allowUnknown: true,
        abortEarly: true,
      },
    }),
    // ... other modules
  ],
})
export class AppModule {}
```

## Accessing Configuration

### Inject ConfigService

```typescript
// src/users/user.service.ts
import { Injectable } from '@nestjs/common';
import { ConfigService } from '@nestjs/config';
import { IAppConfig, IJwtConfig } from '../config';

@Injectable()
export class UserService {
  constructor(private readonly configService: ConfigService) {}

  async initializeService(): Promise<void> {
    // Get entire config
    const appConfig = this.configService.get<IAppConfig>('app');
    console.log(`Running ${appConfig.name} on port ${appConfig.port}`);

    // Get specific value with type safety
    const jwtSecret = this.configService.get<string>('jwt.secret');
    const dbHost = this.configService.get<string>('database.host');

    // Get with default fallback
    const timeout = this.configService.get<number>(
      'request.timeout',
      30000 // default
    );
  }
}
```

### Inject Specific Configurations

```typescript
// src/auth/auth.module.ts
import { Module } from '@nestjs/common';
import { JwtModule } from '@nestjs/jwt';
import { ConfigModule, ConfigService } from '@nestjs/config';
import { IJwtConfig } from '../config';
import { AuthService } from './auth.service';

@Module({
  imports: [
    JwtModule.registerAsync({
      imports: [ConfigModule],
      inject: [ConfigService],
      useFactory: (configService: ConfigService) => {
        const jwtConfig = configService.get<IJwtConfig>('jwt');
        return {
          secret: jwtConfig.secret,
          signOptions: { expiresIn: jwtConfig.expiration },
        };
      },
    }),
  ],
  providers: [AuthService],
})
export class AuthModule {}
```

## Environment-Specific Configuration

### Development (.env.local)

```bash
NODE_ENV=development
APP_NAME=MyApp-Dev
APP_PORT=3000
APP_URL=http://localhost:3000

DATABASE_HOST=localhost
DATABASE_PORT=5432
DATABASE_USERNAME=devuser
DATABASE_PASSWORD=devpass
DATABASE_NAME=myapp_dev

JWT_SECRET=dev-secret-unsafe
JWT_EXPIRATION=24h

LOG_LEVEL=debug
```

### Staging (.env.staging)

```bash
NODE_ENV=staging
APP_NAME=MyApp-Staging
APP_PORT=3000
APP_URL=https://staging.example.com

DATABASE_HOST=staging-db.example.com
DATABASE_PORT=5432
DATABASE_USERNAME=stg_user
DATABASE_PASSWORD=${STAGING_DB_PASSWORD}
DATABASE_NAME=myapp_staging

JWT_SECRET=${STAGING_JWT_SECRET}
JWT_EXPIRATION=24h

LOG_LEVEL=info
```

### Production (.env.production)

```bash
NODE_ENV=production
APP_NAME=MyApp
APP_PORT=3000
APP_URL=https://api.example.com

DATABASE_HOST=prod-db.example.com
DATABASE_PORT=5432
DATABASE_USERNAME=prod_user
DATABASE_PASSWORD=${PRODUCTION_DB_PASSWORD}
DATABASE_NAME=myapp_prod

JWT_SECRET=${PRODUCTION_JWT_SECRET}
JWT_EXPIRATION=12h

LOG_LEVEL=warn
```

## Secrets Management

### Never Commit Secrets

```bash
# ❌ DON'T
git add .env                    # NEVER!
echo "DB_PASSWORD=secret" > .env

# ✅ DO
git add .env.example            # Track template
echo ".env" >> .gitignore       # Ignore actual secrets
echo ".env.*.local" >> .gitignore
```

### Use Environment Variables

```bash
# Local development (machine-specific)
export DATABASE_PASSWORD=local_dev_password
npm run start:dev

# Staging (CI/CD secrets)
DATABASE_PASSWORD=${STAGING_DB_PASSWORD} npm run start:staging

# Production (secure secret management)
# Use: AWS Secrets Manager, HashiCorp Vault, Azure Key Vault, etc.
```

## Setup Checklist

When setting up configuration:

- [ ] Create `/src/config/` directory
- [ ] Define `config.interface.ts` with all config types
- [ ] Create config files for each domain (app, database, jwt, etc.)
- [ ] Create `index.ts` barrel export
- [ ] Create `.env.example` with all required variables
- [ ] Add `.env*` to `.gitignore` (except `.env.example`)
- [ ] Configure `ConfigModule` in `AppModule`
- [ ] Add Joi validation schema
- [ ] Test config loading in different environments
- [ ] Document all configuration variables

## Review Checklist

When reviewing configuration code:

- [ ] All config in `/src/config/` directory
- [ ] Config interfaces defined and typed
- [ ] `.env` files git-ignored (not committed)
- [ ] `.env.example` committed as template
- [ ] Secrets never hardcoded
- [ ] Environment-specific values configurable
- [ ] Joi validation for all variables
- [ ] Default values provided for optional vars
- [ ] Type-safe config access (not `any`)
- [ ] `ConfigModule` registered as global
- [ ] Config injected, not imported directly

## Additional Resources

- For environment file templates and examples, see [environments.md](environments.md)
- For secrets management patterns, see [secrets.md](secrets.md)
