# Swagger Decorator Patterns

This document covers common patterns and conventions for Swagger documentation.

## Authentication Patterns

### Bearer Token

```typescript
@ApiBearerAuth()
@Controller('protected')
export class ProtectedController {
  @Get()
  @ApiBearerAuth()
  protectedEndpoint() {
    // Requires Authorization: Bearer <token>
  }
}
```

### API Key

```typescript
@ApiSecurity('api_key')
@Controller('resources')
export class ResourceController {
  // Setup: DocumentBuilder().addApiKey()
}
```

### OAuth2

```typescript
@ApiOAuth2(['read', 'write'])
@Controller('oauth')
export class OAuthController {
  // Setup: DocumentBuilder().addOAuth2()
}
```

## Response Patterns

### Single Resource

```typescript
@Get(':id')
@ApiResponse({
  status: 200,
  type: UserResponseDto,
})
async getUser(@Param('id') id: string): Promise<UserResponseDto> {
  return this.userService.findById(id);
}
```

### Resource Array

```typescript
@Get()
@ApiResponse({
  status: 200,
  type: [UserResponseDto],
})
async listUsers(): Promise<UserResponseDto[]> {
  return this.userService.findAll();
}
```

### Paginated Response

```typescript
export class PaginatedResponseDto<T> {
  @ApiProperty()
  data: T[];

  @ApiProperty()
  total: number;

  @ApiProperty()
  page: number;

  @ApiProperty()
  limit: number;

  @ApiProperty()
  hasNext: boolean;
}

@Get()
@ApiResponse({
  status: 200,
  schema: {
    properties: {
      data: { type: 'array', items: { $ref: '#/components/schemas/UserResponseDto' } },
      total: { type: 'number' },
      page: { type: 'number' },
      limit: { type: 'number' },
    },
  },
})
async listUsers(
  @Query('page') page: number,
  @Query('limit') limit: number,
): Promise<PaginatedResponseDto<UserResponseDto>> {
  return this.userService.findPaginated({ page, limit });
}
```

### Nested Objects

```typescript
export class OrderResponseDto {
  @ApiProperty()
  id: string;

  @ApiProperty({ type: () => UserResponseDto })
  customer: UserResponseDto;

  @ApiProperty({ type: () => [ProductResponseDto] })
  items: ProductResponseDto[];

  @ApiProperty()
  total: number;
}
```

## Error Response Patterns

### Standard Error Format

```typescript
@ApiResponse({
  status: 400,
  schema: {
    example: {
      statusCode: 400,
      error: 'BadRequest',
      message: 'Email is required',
      timestamp: '2026-04-05T10:30:00.000Z',
      path: '/users',
    },
  },
})
@Post()
async createUser(@Body() dto: CreateUserDto) { }
```

### Multiple Error Responses

```typescript
@ApiResponse({
  status: 201,
  type: UserResponseDto,
  description: 'User created successfully',
})
@ApiResponse({
  status: 400,
  description: 'Invalid input - missing required fields',
  schema: { example: { statusCode: 400, message: 'Email is required' } },
})
@ApiResponse({
  status: 409,
  description: 'Email already in use',
  schema: { example: { statusCode: 409, message: 'Email already in use' } },
})
@Post()
async createUser(@Body() dto: CreateUserDto) { }
```

## Query Parameter Patterns

### Single Query Param

```typescript
@Get()
@ApiQuery({
  name: 'search',
  type: String,
  required: false,
  description: 'Search term for filtering users',
  example: 'john',
})
async searchUsers(@Query('search') search: string) { }
```

### Multiple Query Params

```typescript
@Get()
@ApiQuery({
  name: 'page',
  type: Number,
  required: false,
  description: 'Page number (1-indexed)',
  example: 1,
})
@ApiQuery({
  name: 'limit',
  type: Number,
  required: false,
  description: 'Items per page',
  example: 10,
})
@ApiQuery({
  name: 'sort',
  type: String,
  required: false,
  enum: ['name', 'email', 'createdAt'],
  description: 'Sort field',
  example: 'name',
})
async listUsers(
  @Query('page') page: number = 1,
  @Query('limit') limit: number = 10,
  @Query('sort') sort: string = 'createdAt',
) { }
```

### Array Query Param

```typescript
@Get()
@ApiQuery({
  name: 'roles',
  type: [String],
  description: 'Filter by roles',
  example: ['admin', 'user'],
})
async getUsersByRoles(@Query('roles') roles: string[]) { }
```

## Path Parameter Patterns

### Single ID Parameter

```typescript
@Get(':id')
@ApiParam({
  name: 'id',
  type: String,
  description: 'User unique identifier',
  example: 'usr_123abc',
})
async getUser(@Param('id') id: string) { }
```

### Multiple Parameters

```typescript
@Get(':id/posts/:postId')
@ApiParam({
  name: 'id',
  description: 'User ID',
  example: 'usr_123abc',
})
@ApiParam({
  name: 'postId',
  description: 'Post ID',
  example: 'post_xyz789',
})
async getUserPost(
  @Param('id') id: string,
  @Param('postId') postId: string,
) { }
```

## Request Body Patterns

### Simple DTO

```typescript
@Post()
@ApiResponse({ status: 201, type: UserResponseDto })
async createUser(@Body() dto: CreateUserDto) { }
// Swagger auto-generates from CreateUserDto
```

### Explicit Request Body

```typescript
@Post()
@ApiBody({
  type: CreateUserDto,
  description: 'User creation payload',
  examples: {
    example1: {
      summary: 'Basic user',
      value: {
        email: 'john@example.com',
        name: 'John Doe',
        age: 30,
      },
    },
    example2: {
      summary: 'Another user',
      value: {
        email: 'jane@example.com',
        name: 'Jane Smith',
        age: 25,
      },
    },
  },
})
async createUser(@Body() dto: CreateUserDto) { }
```

## Status Code Patterns

### Create (POST)

```typescript
@Post()
@HttpCode(201)  // Created
@ApiResponse({ status: 201, type: UserResponseDto })
async createUser(@Body() dto: CreateUserDto) { }
```

### Update (PUT/PATCH)

```typescript
@Put(':id')
@HttpCode(200)  // OK
@ApiResponse({ status: 200, type: UserResponseDto })
async updateUser(
  @Param('id') id: string,
  @Body() dto: UpdateUserDto,
) { }
```

### Delete (DELETE)

```typescript
@Delete(':id')
@HttpCode(204)  // No Content
@ApiResponse({ status: 204, description: 'Deleted' })
async deleteUser(@Param('id') id: string) { }
```

## Tag Patterns

### By Resource Type

```typescript
@ApiTags('users')
@Controller('users')
export class UserController { }

@ApiTags('products')
@Controller('products')
export class ProductController { }
```

### Multiple Tags

```typescript
@ApiTags('admin', 'users')
@Controller('admin/users')
export class AdminUserController { }
```

### Tag Descriptions

```typescript
const config = new DocumentBuilder()
  .addTag('users', 'User management operations')
  .addTag('products', 'Product catalog operations')
  .addTag('admin', 'Administrative operations')
  .build();
```

## Field Validation Documentation

### Constraints in ApiProperty

```typescript
export class CreateUserDto {
  @ApiProperty({
    description: 'Email address',
    example: 'john@example.com',
    minLength: 5,
    maxLength: 255,
    pattern: '^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\\.[a-zA-Z]{2,}$',
  })
  @IsEmail()
  email: string;

  @ApiProperty({
    description: 'User age',
    minimum: 18,
    maximum: 120,
  })
  @IsInt()
  @Min(18)
  @Max(120)
  age: number;

  @ApiProperty({
    description: 'User role',
    enum: ['admin', 'user', 'guest'],
    default: 'user',
  })
  @IsEnum(['admin', 'user', 'guest'])
  role: string;
}
```

## Deprecated Endpoint Pattern

```typescript
@Get('/old-endpoint')
@ApiOperation({
  summary: 'Get old data (DEPRECATED)',
  deprecated: true,
  description: 'Use GET /api/v2/data instead',
})
@ApiResponse({
  status: 200,
  deprecated: true,
  schema: { example: { /* ... */ } },
})
async getOldData() {
  // Old implementation
}
```

## Versioning Pattern

```typescript
@ApiTags('users')
@Controller('v1/users')
export class UserControllerV1 {
  @Get()
  async listUsers() { }
}

@ApiTags('users')
@Controller('v2/users')
export class UserControllerV2 {
  @Get()
  async listUsers() { }
  // Different implementation or response format
}
```

Setup in main:
```typescript
const config = new DocumentBuilder()
  .setVersion('2.0')  // Current API version
  .build();
```
