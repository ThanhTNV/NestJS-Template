---
name: documentation
description: Document NestJS APIs with @nestjs/swagger decorators, keeping OpenAPI specs aligned with code. Use DTOs as contract source of truth. Require Swagger docs on all endpoints. Use when adding endpoints or reviewing API contracts.
---

# Documentation

This skill guides implementing comprehensive API documentation with Swagger/OpenAPI decorators in NestJS, keeping specifications synchronized with code.

## Setup

### Installation

```bash
npm install @nestjs/swagger swagger-ui-express
```

### Swagger Module Configuration

```typescript
// src/main.ts
import { SwaggerModule, DocumentBuilder } from '@nestjs/swagger';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);

  // Swagger configuration
  const config = new DocumentBuilder()
    .setTitle('MyApp API')
    .setDescription('Complete API documentation')
    .setVersion('1.0')
    .addBearerAuth()
    .addTag('users', 'User management endpoints')
    .addTag('products', 'Product catalog endpoints')
    .build();

  const document = SwaggerModule.createDocument(app, config);
  SwaggerModule.setup('api', app, document);

  await app.listen(3000);
}

bootstrap();
```

## DTO as Contract Source

### User DTOs (Source of Truth)

```typescript
// src/users/dto/user.dto.ts
import { ApiProperty } from '@nestjs/swagger';
import { IsEmail, IsString, MinLength, IsInt, Min, Max } from 'class-validator';

export class CreateUserDto {
  @ApiProperty({
    description: 'User email address',
    example: 'john@example.com',
  })
  @IsEmail()
  email: string;

  @ApiProperty({
    description: 'User full name',
    example: 'John Doe',
    minLength: 2,
    maxLength: 100,
  })
  @IsString()
  @MinLength(2)
  name: string;

  @ApiProperty({
    description: 'User age',
    example: 30,
    minimum: 0,
    maximum: 150,
  })
  @IsInt()
  @Min(0)
  @Max(150)
  age: number;
}

export class UpdateUserDto {
  @ApiProperty({
    description: 'Updated name (optional)',
    example: 'Jane Doe',
    required: false,
  })
  @IsString()
  @MinLength(2)
  @IsOptional()
  name?: string;

  @ApiProperty({
    description: 'Updated age (optional)',
    example: 31,
    required: false,
  })
  @IsInt()
  @IsOptional()
  age?: number;
}

export class UserResponseDto {
  @ApiProperty({
    description: 'User unique identifier',
    example: 'usr_123abc',
  })
  id: string;

  @ApiProperty({
    description: 'User email address',
    example: 'john@example.com',
  })
  email: string;

  @ApiProperty({
    description: 'User full name',
    example: 'John Doe',
  })
  name: string;

  @ApiProperty({
    description: 'User age',
    example: 30,
  })
  age: number;

  @ApiProperty({
    description: 'Account creation timestamp',
    example: '2026-04-05T10:30:00.000Z',
  })
  createdAt: Date;

  @ApiProperty({
    description: 'Last update timestamp',
    example: '2026-04-05T10:30:00.000Z',
  })
  updatedAt: Date;
}
```

## Endpoint Documentation

### Full Controller with Swagger

```typescript
// src/users/user.controller.ts
import { Controller, Get, Post, Put, Delete, Body, Param, HttpCode } from '@nestjs/common';
import {
  ApiTags,
  ApiOperation,
  ApiResponse,
  ApiParam,
  ApiBearerAuth,
  ApiQuery,
} from '@nestjs/swagger';
import { UserService } from './user.service';
import { CreateUserDto, UpdateUserDto, UserResponseDto } from './dto';

@ApiTags('users')
@ApiBearerAuth()
@Controller('users')
export class UserController {
  constructor(private readonly userService: UserService) {}

  @Get()
  @ApiOperation({
    summary: 'List all users',
    description: 'Retrieve a paginated list of all users in the system.',
  })
  @ApiQuery({
    name: 'page',
    required: false,
    type: Number,
    description: 'Page number (1-indexed)',
    example: 1,
  })
  @ApiQuery({
    name: 'limit',
    required: false,
    type: Number,
    description: 'Items per page',
    example: 10,
  })
  @ApiResponse({
    status: 200,
    description: 'List of users retrieved successfully',
    type: [UserResponseDto],
  })
  @ApiResponse({
    status: 401,
    description: 'Unauthorized - missing or invalid authentication token',
  })
  async listUsers(
    @Query('page') page?: number,
    @Query('limit') limit?: number,
  ): Promise<UserResponseDto[]> {
    return this.userService.findAll({ page, limit });
  }

  @Get(':id')
  @ApiOperation({
    summary: 'Get user by ID',
    description: 'Retrieve a single user by their unique identifier.',
  })
  @ApiParam({
    name: 'id',
    description: 'User unique identifier',
    example: 'usr_123abc',
  })
  @ApiResponse({
    status: 200,
    description: 'User found and returned',
    type: UserResponseDto,
  })
  @ApiResponse({
    status: 404,
    description: 'User not found',
    schema: {
      example: {
        statusCode: 404,
        error: 'NotFound',
        message: 'User not found',
      },
    },
  })
  async getUser(@Param('id') id: string): Promise<UserResponseDto> {
    return this.userService.findById(id);
  }

  @Post()
  @HttpCode(201)
  @ApiOperation({
    summary: 'Create new user',
    description: 'Create a new user account with the provided information.',
  })
  @ApiResponse({
    status: 201,
    description: 'User created successfully',
    type: UserResponseDto,
  })
  @ApiResponse({
    status: 400,
    description: 'Invalid input data',
    schema: {
      example: {
        statusCode: 400,
        message: ['email must be an email'],
        error: 'Bad Request',
      },
    },
  })
  @ApiResponse({
    status: 409,
    description: 'Email already in use',
    schema: {
      example: {
        statusCode: 409,
        error: 'Conflict',
        message: 'Email already in use',
      },
    },
  })
  async createUser(@Body() createUserDto: CreateUserDto): Promise<UserResponseDto> {
    return this.userService.create(createUserDto);
  }

  @Put(':id')
  @ApiOperation({
    summary: 'Update user',
    description: 'Update specific fields of an existing user.',
  })
  @ApiParam({
    name: 'id',
    description: 'User unique identifier',
    example: 'usr_123abc',
  })
  @ApiResponse({
    status: 200,
    description: 'User updated successfully',
    type: UserResponseDto,
  })
  @ApiResponse({
    status: 404,
    description: 'User not found',
  })
  async updateUser(
    @Param('id') id: string,
    @Body() updateUserDto: UpdateUserDto,
  ): Promise<UserResponseDto> {
    return this.userService.update(id, updateUserDto);
  }

  @Delete(':id')
  @HttpCode(204)
  @ApiOperation({
    summary: 'Delete user',
    description: 'Permanently delete a user account.',
  })
  @ApiParam({
    name: 'id',
    description: 'User unique identifier',
    example: 'usr_123abc',
  })
  @ApiResponse({
    status: 204,
    description: 'User deleted successfully',
  })
  @ApiResponse({
    status: 404,
    description: 'User not found',
  })
  async deleteUser(@Param('id') id: string): Promise<void> {
    return this.userService.delete(id);
  }
}
```

## ApiProperty Decorator Reference

### Common Properties

```typescript
@ApiProperty({
  // Description shown in docs
  description: 'User email address',

  // Example value shown in Swagger UI
  example: 'john@example.com',

  // Mark as required/optional
  required: true,
  isArray: false,

  // Type and format
  type: String,
  format: 'email',

  // Validation constraints
  minLength: 5,
  maxLength: 255,
  minimum: 0,
  maximum: 100,
  pattern: '^[a-zA-Z0-9]+$',

  // For enums
  enum: ['active', 'inactive', 'pending'],
  enumName: 'UserStatus',

  // Default value
  default: 'active',

  // Deprecation notice
  deprecated: false,

  // For relationships
  type: () => UserResponseDto,
  isArray: true,
})
field: string;
```

## Error Response Documentation

```typescript
// src/common/dto/error-response.dto.ts
import { ApiProperty } from '@nestjs/swagger';

export class ErrorResponseDto {
  @ApiProperty({
    description: 'HTTP status code',
    example: 400,
  })
  statusCode: number;

  @ApiProperty({
    description: 'Error type',
    example: 'BadRequest',
  })
  error: string;

  @ApiProperty({
    description: 'Error message',
    example: 'Email is required',
  })
  message: string;

  @ApiProperty({
    description: 'Request timestamp',
    example: '2026-04-05T10:30:00.000Z',
  })
  timestamp: string;

  @ApiProperty({
    description: 'Request path',
    example: '/users',
  })
  path: string;
}
```

## Validation Checklist

When documenting endpoints:

- [ ] All endpoints have `@ApiOperation` with summary and description
- [ ] All request DTOs documented with `@ApiProperty` on each field
- [ ] All response DTOs documented with `@ApiProperty` on each field
- [ ] Error responses documented with status codes and examples
- [ ] Query parameters documented with `@ApiQuery`
- [ ] Path parameters documented with `@ApiParam`
- [ ] Authentication requirements marked with `@ApiBearerAuth`
- [ ] HTTP status codes accurate (201 for POST, 204 for DELETE, etc.)
- [ ] Example values realistic and match actual API behavior
- [ ] Tags match endpoint groupings
- [ ] Optional fields marked with `required: false`
- [ ] Deprecated endpoints marked with `deprecated: true`

## Generation Checklist

When adding new endpoints:

- [ ] Create/update DTOs with `@ApiProperty` on all fields
- [ ] Add `@ApiTags` to controller
- [ ] Add `@ApiOperation` to endpoint method
- [ ] Add `@ApiResponse` for success case
- [ ] Add `@ApiResponse` for each error case
- [ ] Document all `@ApiParam` and `@ApiQuery`
- [ ] Add `@ApiBearerAuth` if authentication required
- [ ] Set correct HTTP status codes
- [ ] Include realistic examples
- [ ] Test in Swagger UI
- [ ] Validate OpenAPI schema generation

## Review Checklist

When reviewing API documentation:

- [ ] Every endpoint has descriptive `@ApiOperation`
- [ ] All DTOs fully documented with `@ApiProperty`
- [ ] No missing error response documentation
- [ ] HTTP status codes are correct
- [ ] Examples are realistic
- [ ] No hardcoded values leaked in descriptions
- [ ] Tags are consistent
- [ ] Authentication marked where required
- [ ] API versioning clear
- [ ] Swagger UI accessible and readable

## Additional Resources

- For sync validation strategies, see [sync.md](sync.md)
- For decorator patterns and examples, see [patterns.md](patterns.md)
