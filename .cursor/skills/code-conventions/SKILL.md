---
name: code-conventions
description: Enforce NestJS naming conventions, folder structure, DTO usage, and linting rules. Ensure consistency across controllers, services, repositories, and modules. Use when generating code, reviewing code structure, or auditing project consistency.
---

# Code Conventions

This skill enforces coding standards for NestJS projects to maintain consistency and quality. Use it when writing new code or reviewing existing code for compliance.

## Quick Reference

| Artifact | Pattern | Location |
|----------|---------|----------|
| **Controller** | `<name>.controller.ts` | `src/<feature>/` |
| **Service** | `<name>.service.ts` | `src/<feature>/` |
| **Repository** | `<name>.repository.ts` | `src/<feature>/` |
| **Module** | `<name>.module.ts` | `src/<feature>/` |
| **DTO** | `create-<name>.dto.ts`, `update-<name>.dto.ts` | `src/<feature>/dto/` |
| **Entity** | `<name>.entity.ts` | `src/<feature>/entities/` |
| **Unit Test** | `<name>.spec.ts` | colocated with source |

## Naming Conventions

### File Names

```typescript
// ✅ GOOD
user.controller.ts
user.service.ts
user.repository.ts
create-user.dto.ts
user.entity.ts
user.module.ts

// ❌ BAD
UserController.ts
userService.ts
user_repository.ts
UserDTO.ts
User.ts
```

### Class Names

```typescript
// ✅ GOOD
export class UserController { }
export class UserService { }
export class UserRepository { }
export class CreateUserDto { }
export class User { }

// ❌ BAD
export class user_controller { }
export class UserServ { }
export class repo_user { }
export class createUserDTO { }
```

### Method Names

```typescript
// ✅ GOOD - Services/Repositories
findAll()
findById(id)
create(dto)
update(id, dto)
remove(id)

// ✅ GOOD - Controllers
@Get()
getAll()

@Get(':id')
getById(@Param('id') id)

@Post()
create(@Body() dto)

@Put(':id')
update(@Param('id') id, @Body() dto)

@Delete(':id')
remove(@Param('id') id)
```

## Layering Rules

### Controllers

```typescript
// ✅ GOOD - Thin controller, delegates to service
@Controller('users')
export class UserController {
  constructor(private readonly userService: UserService) {}

  @Get()
  async getAll() {
    return this.userService.findAll();
  }

  @Post()
  async create(@Body() dto: CreateUserDto) {
    return this.userService.create(dto);
  }
}

// ❌ BAD - Business logic in controller
@Controller('users')
export class UserController {
  @Post()
  async create(@Body() dto: CreateUserDto) {
    const user = new User();
    user.name = dto.name;
    // DB call in controller
    await this.db.query('INSERT INTO users...');
    return user;
  }
}
```

### Services

```typescript
// ✅ GOOD - Business logic, delegates to repository
@Injectable()
export class UserService {
  constructor(private readonly userRepository: UserRepository) {}

  async create(dto: CreateUserDto) {
    // Business logic here
    const user = new User();
    user.name = dto.name;
    // Delegate DB interaction to repository
    return this.userRepository.save(user);
  }
}

// ❌ BAD - Repository calls in controller
@Controller('users')
export class UserController {
  constructor(private readonly userRepository: UserRepository) {}

  @Post()
  async create(@Body() dto: CreateUserDto) {
    return this.userRepository.save(dto);
  }
}
```

### Repositories

```typescript
// ✅ GOOD - Only DB interactions
@Injectable()
export class UserRepository {
  constructor(private readonly db: Database) {}

  async save(user: User): Promise<User> {
    return this.db.query('INSERT INTO users...', user);
  }

  async findById(id: string): Promise<User | null> {
    return this.db.query('SELECT * FROM users WHERE id = ?', id);
  }
}

// ❌ BAD - Business logic in repository
@Injectable()
export class UserRepository {
  async save(user: User): Promise<User> {
    // Validating business rules in repository
    if (user.age < 18) throw new Error('Too young');
    return this.db.query('INSERT INTO users...', user);
  }
}
```

## DTO Usage

### Create DTOs for All Input

```typescript
// ✅ GOOD - Explicit DTO validation
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

@Post()
async create(@Body() dto: CreateUserDto) {
  return this.userService.create(dto);
}

// ❌ BAD - No validation
@Post()
async create(@Body() body: any) {
  return this.userService.create(body);
}
```

### Create Separate DTOs for Updates

```typescript
// ✅ GOOD
export class CreateUserDto {
  @IsString()
  @IsNotEmpty()
  name: string;

  @IsEmail()
  email: string;
}

export class UpdateUserDto {
  @IsString()
  @IsOptional()
  name?: string;

  @IsEmail()
  @IsOptional()
  email?: string;
}

// ❌ BAD - Reusing create DTO for updates
@Put(':id')
async update(@Param('id') id: string, @Body() dto: CreateUserDto) {
  // Required fields in create DTO now optional for update
}
```

## Linting & Formatting

### ESLint Enforcement

Run before commits:

```bash
npm run lint
npm run lint:fix
```

### Prettier Formatting

Ensure all code is formatted:

```bash
npm run format
npm run format:check
```

### Pre-commit Validation

Use the validation script to check compliance:

```bash
npm run validate:conventions
```

See [validation.md](validation.md) for details.

## Module Structure

```typescript
// ✅ GOOD - Complete feature module
@Module({
  controllers: [UserController],
  providers: [UserService, UserRepository],
  exports: [UserService],  // Export only what's needed
})
export class UserModule {}

// ❌ BAD - No explicit boundaries
@Module({
  controllers: [UserController],
  providers: [UserService, UserRepository, OtherService, AnotherService],
})
export class UserModule {}
```

## Common Mistakes Checklist

- [ ] Database calls in controllers (should be in repositories)
- [ ] Business logic in repositories (should be in services)
- [ ] Missing DTOs for request validation
- [ ] Using `@Body() body: any` instead of typed DTO
- [ ] File names in PascalCase instead of kebab-case
- [ ] Mixing concerns in a single file
- [ ] Not exporting needed items from modules
- [ ] HTTP logic in services

## Code Review Workflow

When reviewing code for consistency:

1. **Check file naming**: Are all files kebab-case?
2. **Verify structure**: Is code in the right layer (controller/service/repository)?
3. **Validate DTOs**: Are all inputs validated with DTOs?
4. **Confirm layering**: No DB calls in controllers? No business logic in repositories?
5. **Run linters**: Do `npm run lint` and `npm run format:check` pass?
6. **Validate project**: Run `npm run validate:conventions` for structure checks

## Code Generation Workflow

When generating new code:

1. **Create the module folder**: `src/<feature>/`
2. **Generate controller** with routing
3. **Generate service** with business logic
4. **Generate repository** with DB interactions
5. **Generate DTOs** for input/output
6. **Generate entity** for data model
7. **Create module** with proper exports/imports
8. **Add unit tests** colocated as `.spec.ts`
9. **Run linters** to ensure formatting
10. **Register module** in root `AppModule`

## Additional Resources

- For detailed structure guidelines, see [structure.md](structure.md)
- For project architecture rules, refer to `.cursor/rules/nestjs-standards.mdc`
