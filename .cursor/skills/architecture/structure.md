# Architecture Structure

This document provides detailed guidance on implementing layered architecture within NestJS feature modules.

## Feature Module Blueprint

```
src/users/
├── users.module.ts                    # Module declaration (glue)
├── users.controller.ts                # Presentation layer
├── users.controller.spec.ts
├── users.service.ts                   # Application layer
├── users.service.spec.ts
├── users.repository.ts                # Infrastructure layer
├── users.repository.spec.ts
├── dto/                               # Infrastructure layer (contracts)
│   ├── create-user.dto.ts
│   └── update-user.dto.ts
└── entities/                          # Domain layer
    └── user.entity.ts
```

## Layer Implementation Examples

### Domain Layer: Entity

Entities represent pure domain concepts. They contain only business rules relevant to the domain.

```typescript
// users/entities/user.entity.ts
export class User {
  id: string;
  email: string;
  name: string;
  age: number;
  createdAt: Date;

  // Pure domain logic - no persistence concerns
  isAdult(): boolean {
    return this.age >= 18;
  }

  canPurchase(): boolean {
    return this.isAdult();
  }

  // ❌ DON'T - No decorators (except domain enums/validators)
  // @Column()
  // @Exclude()

  // ❌ DON'T - No persistence awareness
  // async save(): Promise<void> { }

  // ❌ DON'T - No HTTP awareness
  // @ApiProperty()
}
```

### Infrastructure Layer: DTO

DTOs define the contract between external clients and your API. They handle input/output validation.

```typescript
// users/dto/create-user.dto.ts
import { IsString, IsEmail, IsInt, Min, Max, IsNotEmpty } from 'class-validator';
import { ApiProperty } from '@nestjs/swagger';

export class CreateUserDto {
  @ApiProperty({ description: 'User full name' })
  @IsString()
  @IsNotEmpty()
  name: string;

  @ApiProperty({ description: 'User email address' })
  @IsEmail()
  email: string;

  @ApiProperty({ description: 'User age', minimum: 0, maximum: 150 })
  @IsInt()
  @Min(0)
  @Max(150)
  age: number;
}

// users/dto/update-user.dto.ts
import { PartialType } from '@nestjs/swagger';

export class UpdateUserDto extends PartialType(CreateUserDto) {
  // All fields optional for updates
}
```

### Infrastructure Layer: Repository

Repositories handle all persistence concerns. They translate between domain entities and persistence format.

```typescript
// users/users.repository.ts
import { Injectable } from '@nestjs/common';
import { User } from './entities/user.entity';

@Injectable()
export class UsersRepository {
  constructor(
    private readonly database: Database,
    // or: private readonly orm: getRepository(User)
  ) {}

  // Pure data access - no business logic
  async findAll(): Promise<User[]> {
    const records = await this.database.query(
      'SELECT * FROM users ORDER BY created_at DESC',
    );
    return records.map(record => this.mapToDomain(record));
  }

  async findById(id: string): Promise<User | null> {
    const record = await this.database.query(
      'SELECT * FROM users WHERE id = ?',
      [id],
    );
    return record ? this.mapToDomain(record) : null;
  }

  async findByEmail(email: string): Promise<User | null> {
    const record = await this.database.query(
      'SELECT * FROM users WHERE email = ?',
      [email],
    );
    return record ? this.mapToDomain(record) : null;
  }

  async save(user: User): Promise<User> {
    const record = await this.database.query(
      'INSERT INTO users (id, email, name, age) VALUES (?, ?, ?, ?)',
      [user.id, user.email, user.name, user.age],
    );
    return this.mapToDomain(record);
  }

  async update(id: string, user: Partial<User>): Promise<User> {
    const fields = Object.entries(user)
      .filter(([_, value]) => value !== undefined)
      .map(([key]) => `${key} = ?`)
      .join(', ');

    const values = Object.values(user).filter(v => v !== undefined);

    const record = await this.database.query(
      `UPDATE users SET ${fields} WHERE id = ?`,
      [...values, id],
    );
    return this.mapToDomain(record);
  }

  async delete(id: string): Promise<void> {
    await this.database.query('DELETE FROM users WHERE id = ?', [id]);
  }

  // ❌ DON'T - Business logic
  // async saveWithValidation(user: User): Promise<User> {
  //   if (user.age < 18) throw new Error('Too young');
  // }

  // ❌ DON'T - External calls (unless data-fetching)
  // async saveAndNotify(user: User): Promise<User> {
  //   const saved = await this.save(user);
  //   await this.emailService.sendWelcome(user.email);
  // }

  private mapToDomain(record: any): User {
    const user = new User();
    user.id = record.id;
    user.email = record.email;
    user.name = record.name;
    user.age = record.age;
    user.createdAt = new Date(record.created_at);
    return user;
  }
}
```

### Application Layer: Service

Services contain all business logic and orchestrate repositories and external services.

```typescript
// users/users.service.ts
import { Injectable, BadRequestException } from '@nestjs/common';
import { UsersRepository } from './users.repository';
import { User } from './entities/user.entity';
import { CreateUserDto } from './dto/create-user.dto';
import { UpdateUserDto } from './dto/update-user.dto';

@Injectable()
export class UsersService {
  constructor(
    private readonly usersRepository: UsersRepository,
    private readonly emailService: EmailService, // External service
  ) {}

  // Business logic: Orchestration + validation
  async createUser(dto: CreateUserDto): Promise<User> {
    // Validate business rules
    const existing = await this.usersRepository.findByEmail(dto.email);
    if (existing) {
      throw new BadRequestException('Email already in use');
    }

    // Create domain object
    const user = new User();
    user.id = this.generateId();
    user.email = dto.email;
    user.name = dto.name;
    user.age = dto.age;
    user.createdAt = new Date();

    // Persist
    const saved = await this.usersRepository.save(user);

    // Side effects (external services)
    try {
      await this.emailService.sendWelcome(saved.email);
    } catch (error) {
      // Handle gracefully - user was created even if email fails
      console.error('Welcome email failed:', error);
    }

    return saved;
  }

  async updateUser(id: string, dto: UpdateUserDto): Promise<User> {
    // Load current state
    const user = await this.usersRepository.findById(id);
    if (!user) {
      throw new NotFoundException('User not found');
    }

    // Apply business rules to changes
    if (dto.email && dto.email !== user.email) {
      const existing = await this.usersRepository.findByEmail(dto.email);
      if (existing) {
        throw new BadRequestException('Email already in use');
      }
      user.email = dto.email;
    }

    if (dto.name !== undefined) user.name = dto.name;
    if (dto.age !== undefined) {
      if (dto.age < 0 || dto.age > 150) {
        throw new BadRequestException('Invalid age');
      }
      user.age = dto.age;
    }

    // Persist
    return this.usersRepository.update(id, user);
  }

  async getUserById(id: string): Promise<User> {
    const user = await this.usersRepository.findById(id);
    if (!user) {
      throw new NotFoundException('User not found');
    }
    return user;
  }

  async listUsers(limit: number = 10): Promise<User[]> {
    return this.usersRepository.findAll();
  }

  async deleteUser(id: string): Promise<void> {
    const user = await this.getUserById(id);
    
    // Audit trail
    await this.auditService.log('user_deleted', user.id);
    
    // Delete
    await this.usersRepository.delete(id);
  }

  // Pure business logic helper
  private generateId(): string {
    return `user_${Date.now()}_${Math.random().toString(36).substr(2, 9)}`;
  }

  // ❌ DON'T - HTTP concerns
  // async getUser(id: string): Promise<HttpResponse> {
  //   const user = await this.usersRepository.findById(id);
  //   return { statusCode: 200, data: user };
  // }

  // ❌ DON'T - Database queries
  // async createWithDb(dto: CreateUserDto): Promise<User> {
  //   return this.db.query('INSERT INTO users...', dto);
  // }
}
```

### Presentation Layer: Controller

Controllers handle HTTP concerns only. They parse requests, validate (via DTOs), and delegate to services.

```typescript
// users/users.controller.ts
import {
  Controller,
  Get,
  Post,
  Put,
  Delete,
  Body,
  Param,
  HttpCode,
  HttpStatus,
  Query,
} from '@nestjs/common';
import { ApiTags, ApiOperation, ApiResponse } from '@nestjs/swagger';
import { UsersService } from './users.service';
import { User } from './entities/user.entity';
import { CreateUserDto } from './dto/create-user.dto';
import { UpdateUserDto } from './dto/update-user.dto';

@ApiTags('users')
@Controller('users')
export class UsersController {
  constructor(private readonly usersService: UsersService) {}

  @Get()
  @ApiOperation({ summary: 'List all users' })
  @ApiResponse({ status: 200, description: 'Users list', type: [User] })
  async listUsers(@Query('limit') limit?: string): Promise<User[]> {
    return this.usersService.listUsers(parseInt(limit || '10'));
  }

  @Get(':id')
  @ApiOperation({ summary: 'Get user by ID' })
  @ApiResponse({ status: 200, description: 'User found', type: User })
  @ApiResponse({ status: 404, description: 'User not found' })
  async getUser(@Param('id') id: string): Promise<User> {
    return this.usersService.getUserById(id);
  }

  @Post()
  @HttpCode(HttpStatus.CREATED)
  @ApiOperation({ summary: 'Create new user' })
  @ApiResponse({ status: 201, description: 'User created', type: User })
  async createUser(@Body() createUserDto: CreateUserDto): Promise<User> {
    return this.usersService.createUser(createUserDto);
  }

  @Put(':id')
  @ApiOperation({ summary: 'Update user' })
  @ApiResponse({ status: 200, description: 'User updated', type: User })
  async updateUser(
    @Param('id') id: string,
    @Body() updateUserDto: UpdateUserDto,
  ): Promise<User> {
    return this.usersService.updateUser(id, updateUserDto);
  }

  @Delete(':id')
  @HttpCode(HttpStatus.NO_CONTENT)
  @ApiOperation({ summary: 'Delete user' })
  @ApiResponse({ status: 204, description: 'User deleted' })
  async deleteUser(@Param('id') id: string): Promise<void> {
    return this.usersService.deleteUser(id);
  }

  // ❌ DON'T - Business logic
  // async createUser(@Body() dto: CreateUserDto) {
  //   if (dto.age < 18) throw new Error('Too young');
  //   return this.usersService.save(dto);
  // }

  // ❌ DON'T - Database queries
  // async getUser(@Param('id') id: string) {
  //   return this.db.query('SELECT * FROM users WHERE id = ?', [id]);
  // }
}
```

### Module: Glue Layer

Modules wire dependencies and expose what's needed by other modules.

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

## Cross-Module Dependencies

### Email Service (External)

```typescript
// emails/emails.service.ts
@Injectable()
export class EmailService {
  async sendWelcome(email: string): Promise<void> {
    // Send email
  }
}

// emails/emails.module.ts
@Module({
  providers: [EmailService],
  exports: [EmailService],
})
export class EmailsModule {}

// users/users.module.ts
import { EmailModule } from '../emails/emails.module';

@Module({
  imports: [EmailsModule],
  controllers: [UsersController],
  providers: [UsersService, UsersRepository],
  exports: [UsersService],
})
export class UsersModule {}

// users/users.service.ts - Inject exported service
@Injectable()
export class UsersService {
  constructor(
    private readonly usersRepository: UsersRepository,
    private readonly emailService: EmailService, // From EmailsModule.exports
  ) {}
}
```

## Common Structure Mistakes

| ❌ Bad | ✅ Good | Why |
|--------|---------|-----|
| DB queries in controllers | Queries in repository | Separation of concerns |
| Business logic in repository | Logic in service | Testability, reusability |
| HTTP exceptions in service | HTTP exceptions in controller | Service not HTTP-aware |
| Repositories exported | Only services exported | Clear module boundaries |
| Circular module dependencies | Acyclic dependency graph | Testability, maintainability |
| Multiple concerns per file | One concern per file | Clarity, reusability |
| Importing internal classes | Using module exports only | Stability, loose coupling |
