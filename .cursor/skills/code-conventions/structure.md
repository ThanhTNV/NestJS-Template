# Project Structure

This document provides detailed guidance on the folder and file structure for NestJS projects following this skill.

## Root Structure

```
src/
  common/              # Shared utilities
    decorators/
    filters/
    guards/
    interceptors/
    pipes/
    utils/
  config/              # Configuration
    database.config.ts
    app.config.ts
  <feature>/           # Feature modules
    <feature>.module.ts
    <feature>.controller.ts
    <feature>.service.ts
    <feature>.repository.ts
    dto/
      create-<feature>.dto.ts
      update-<feature>.dto.ts
    entities/
      <feature>.entity.ts
  app.module.ts
  main.ts

test/                  # E2e tests only
  <feature>.e2e-spec.ts
```

## Feature Module Structure

Each feature (users, products, orders, etc.) gets its own folder with clear separation of concerns:

```
src/users/
  users.module.ts          # Module declaration
  users.controller.ts      # HTTP layer
  users.controller.spec.ts # Controller unit tests
  users.service.ts         # Business logic
  users.service.spec.ts    # Service unit tests
  users.repository.ts      # Data layer
  users.repository.spec.ts # Repository unit tests
  dto/
    create-user.dto.ts
    update-user.dto.ts
  entities/
    user.entity.ts
```

### Module File

```typescript
// users/users.module.ts
import { Module } from '@nestjs/common';
import { UsersController } from './users.controller';
import { UsersService } from './users.service';
import { UsersRepository } from './users.repository';

@Module({
  controllers: [UsersController],
  providers: [UsersService, UsersRepository],
  exports: [UsersService], // Only export what other modules need
})
export class UsersModule {}
```

### Controller File

```typescript
// users/users.controller.ts
import { Controller, Get, Post, Body, Param } from '@nestjs/common';
import { UsersService } from './users.service';
import { CreateUserDto } from './dto/create-user.dto';

@Controller('users')
export class UsersController {
  constructor(private readonly usersService: UsersService) {}

  @Get()
  findAll() {
    return this.usersService.findAll();
  }

  @Post()
  create(@Body() createUserDto: CreateUserDto) {
    return this.usersService.create(createUserDto);
  }
}
```

### Service File

```typescript
// users/users.service.ts
import { Injectable } from '@nestjs/common';
import { UsersRepository } from './users.repository';
import { CreateUserDto } from './dto/create-user.dto';

@Injectable()
export class UsersService {
  constructor(private readonly usersRepository: UsersRepository) {}

  findAll() {
    return this.usersRepository.findAll();
  }

  create(createUserDto: CreateUserDto) {
    return this.usersRepository.save(createUserDto);
  }
}
```

### Repository File

```typescript
// users/users.repository.ts
import { Injectable } from '@nestjs/common';
import { User } from './entities/user.entity';

@Injectable()
export class UsersRepository {
  async findAll(): Promise<User[]> {
    // Database query
  }

  async save(user: User): Promise<User> {
    // Database save
  }
}
```

### DTO Files

```typescript
// users/dto/create-user.dto.ts
import { IsString, IsEmail, IsInt, Min } from 'class-validator';

export class CreateUserDto {
  @IsString()
  @IsNotEmpty()
  name: string;

  @IsEmail()
  email: string;

  @IsInt()
  @Min(0)
  age: number;
}

// users/dto/update-user.dto.ts
import { IsString, IsEmail, IsOptional } from 'class-validator';

export class UpdateUserDto {
  @IsString()
  @IsOptional()
  name?: string;

  @IsEmail()
  @IsOptional()
  email?: string;
}
```

### Entity File

```typescript
// users/entities/user.entity.ts
export class User {
  id: string;
  name: string;
  email: string;
  age: number;
}
```

## Common/ Folder

Shared utilities that multiple modules use:

```
src/common/
  decorators/
    current-user.decorator.ts
  filters/
    http-exception.filter.ts
  guards/
    auth.guard.ts
    roles.guard.ts
  interceptors/
    logging.interceptor.ts
  pipes/
    validation.pipe.ts
  utils/
    logger.ts
    helpers.ts
```

### Example: Custom Decorator

```typescript
// src/common/decorators/current-user.decorator.ts
import { createParamDecorator, ExecutionContext } from '@nestjs/common';

export const CurrentUser = createParamDecorator(
  (data: unknown, ctx: ExecutionContext) => {
    const request = ctx.switchToHttp().getRequest();
    return request.user;
  },
);
```

## Config/ Folder

Centralized configuration:

```
src/config/
  database.config.ts
  app.config.ts
```

### Example: Database Config

```typescript
// src/config/database.config.ts
import { registerAs } from '@nestjs/config';

export default registerAs('database', () => ({
  host: process.env.DB_HOST || 'localhost',
  port: parseInt(process.env.DB_PORT || '5432'),
  username: process.env.DB_USERNAME || 'user',
  password: process.env.DB_PASSWORD,
  database: process.env.DB_NAME || 'myapp',
}));
```

## Test Files

Unit tests are colocated with source files:

```
src/users/users.service.ts
src/users/users.service.spec.ts  # Unit test

src/users/users.controller.ts
src/users/users.controller.spec.ts  # Unit test

test/users.e2e-spec.ts  # E2e test
```

### Example Unit Test

```typescript
// users/users.service.spec.ts
import { Test, TestingModule } from '@nestjs/testing';
import { UsersService } from './users.service';
import { UsersRepository } from './users.repository';

describe('UsersService', () => {
  let service: UsersService;
  let repository: UsersRepository;

  beforeEach(async () => {
    const module: TestingModule = await Test.createTestingModule({
      providers: [
        UsersService,
        {
          provide: UsersRepository,
          useValue: { findAll: jest.fn() },
        },
      ],
    }).compile();

    service = module.get<UsersService>(UsersService);
    repository = module.get<UsersRepository>(UsersRepository);
  });

  it('should return all users', async () => {
    const users = [{ id: '1', name: 'John' }];
    jest.spyOn(repository, 'findAll').mockResolvedValue(users);

    expect(await service.findAll()).toEqual(users);
  });
});
```

## Do's and Don'ts

| ✅ Do | ❌ Don't |
|------|---------|
| Keep each module focused on one feature | Mix multiple features in one module |
| Colocate related files (`*.service.ts`, `*.service.spec.ts`) | Scatter related files across folders |
| Export only what's needed from modules | Export everything |
| Use DTOs for all input/output | Pass raw objects around |
| Put business logic in services | Put business logic in controllers or repositories |
| Keep repositories database-only | Mix business logic into repositories |
| One class per file (except DTOs) | Multiple classes per file |
