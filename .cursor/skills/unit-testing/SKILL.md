---
name: unit-testing
description: Write effective unit tests for NestJS services, repositories, and controllers using Jest. Enforce dependency mocking, coverage thresholds, and testing best practices. Use when generating test files, reviewing test quality, or ensuring adequate test coverage.
---

# Unit Testing

This skill guides writing comprehensive unit tests for NestJS projects using Jest and the @nestjs/testing module. Enforce mocking, coverage, and best practices.

## Quick Reference

| Artifact | Min Coverage | File Pattern | Key Pattern |
|----------|--------------|--------------|-------------|
| **Service** | 80% | `*.service.spec.ts` | Mock dependencies, test business logic |
| **Repository** | 85% | `*.repository.spec.ts` | Mock database, test queries |
| **Controller** | 70% | `*.controller.spec.ts` | Mock service, test HTTP handling |
| **Pipe/Guard** | 80% | `*.pipe.spec.ts`, `*.guard.spec.ts` | Mock inputs, test conditions |

## Coverage Thresholds

```bash
# Project-wide targets
npm test -- --coverage

# Enforce in CI with:
npm test -- --coverage --coverageThreshold='{
  "global": {
    "branches": 75,
    "functions": 80,
    "lines": 80,
    "statements": 80
  }
}'
```

## Test Setup Pattern

### Service Test Template

```typescript
// user.service.spec.ts
import { Test, TestingModule } from '@nestjs/testing';
import { UserService } from './user.service';
import { UserRepository } from './user.repository';
import { ConflictException, NotFoundException } from '@nestjs/common';

describe('UserService', () => {
  let service: UserService;
  let repository: UserRepository;

  // Setup mock repository
  const mockUserRepository = {
    findById: jest.fn(),
    findByEmail: jest.fn(),
    save: jest.fn(),
    update: jest.fn(),
    delete: jest.fn(),
  };

  beforeEach(async () => {
    // Create test module
    const module: TestingModule = await Test.createTestingModule({
      providers: [
        UserService,
        {
          provide: UserRepository,
          useValue: mockUserRepository,
        },
      ],
    }).compile();

    // Get instances
    service = module.get<UserService>(UserService);
    repository = module.get<UserRepository>(UserRepository);

    // Clear mocks before each test
    jest.clearAllMocks();
  });

  describe('createUser', () => {
    it('should create a user', async () => {
      const dto = { email: 'test@test.com', name: 'John', age: 30 };
      const created = { id: '1', ...dto, createdAt: new Date() };

      // Arrange
      mockUserRepository.findByEmail.mockResolvedValue(null);
      mockUserRepository.save.mockResolvedValue(created);

      // Act
      const result = await service.createUser(dto);

      // Assert
      expect(result).toEqual(created);
      expect(mockUserRepository.findByEmail).toHaveBeenCalledWith(dto.email);
      expect(mockUserRepository.save).toHaveBeenCalledWith(expect.objectContaining({
        email: dto.email,
        name: dto.name,
      }));
    });

    it('should reject duplicate email', async () => {
      const dto = { email: 'test@test.com', name: 'John', age: 30 };

      // Arrange
      mockUserRepository.findByEmail.mockResolvedValue({ id: '1', ...dto });

      // Act & Assert
      await expect(service.createUser(dto))
        .rejects.toThrow(ConflictException);

      expect(mockUserRepository.save).not.toHaveBeenCalled();
    });

    it('should reject invalid age', async () => {
      const dto = { email: 'test@test.com', name: 'John', age: 150 };

      // Act & Assert
      await expect(service.createUser(dto))
        .rejects.toThrow(BadRequestException);

      expect(mockUserRepository.findByEmail).not.toHaveBeenCalled();
    });
  });

  describe('getUserById', () => {
    it('should return user if found', async () => {
      const user = { id: '1', email: 'test@test.com', name: 'John' };

      // Arrange
      mockUserRepository.findById.mockResolvedValue(user);

      // Act
      const result = await service.getUserById('1');

      // Assert
      expect(result).toEqual(user);
      expect(mockUserRepository.findById).toHaveBeenCalledWith('1');
    });

    it('should throw NotFoundException if not found', async () => {
      // Arrange
      mockUserRepository.findById.mockResolvedValue(null);

      // Act & Assert
      await expect(service.getUserById('999'))
        .rejects.toThrow(NotFoundException);
    });
  });
});
```

### Repository Test Template

```typescript
// user.repository.spec.ts
import { Test, TestingModule } from '@nestjs/testing';
import { UserRepository } from './user.repository';
import { User } from './entities/user.entity';

describe('UserRepository', () => {
  let repository: UserRepository;

  const mockDatabase = {
    query: jest.fn(),
  };

  beforeEach(async () => {
    const module: TestingModule = await Test.createTestingModule({
      providers: [
        UserRepository,
        {
          provide: 'Database',
          useValue: mockDatabase,
        },
      ],
    }).compile();

    repository = module.get<UserRepository>(UserRepository);
    jest.clearAllMocks();
  });

  describe('findById', () => {
    it('should return user by ID', async () => {
      const record = {
        id: '1',
        email: 'test@test.com',
        name: 'John',
        age: 30,
        created_at: new Date(),
      };

      // Arrange
      mockDatabase.query.mockResolvedValue(record);

      // Act
      const result = await repository.findById('1');

      // Assert
      expect(result).toEqual(expect.objectContaining({
        id: '1',
        email: 'test@test.com',
      }));
      expect(mockDatabase.query).toHaveBeenCalledWith(
        expect.stringContaining('SELECT * FROM users WHERE id = ?'),
        ['1']
      );
    });

    it('should return null if user not found', async () => {
      // Arrange
      mockDatabase.query.mockResolvedValue(null);

      // Act
      const result = await repository.findById('999');

      // Assert
      expect(result).toBeNull();
    });
  });

  describe('save', () => {
    it('should save user and return created record', async () => {
      const user = new User();
      user.id = '1';
      user.email = 'test@test.com';
      user.name = 'John';
      user.age = 30;

      const saved = { ...user, created_at: new Date() };

      // Arrange
      mockDatabase.query.mockResolvedValue(saved);

      // Act
      const result = await repository.save(user);

      // Assert
      expect(result).toEqual(expect.objectContaining({
        id: '1',
        email: 'test@test.com',
      }));
      expect(mockDatabase.query).toHaveBeenCalled();
    });
  });
});
```

### Controller Test Template

```typescript
// user.controller.spec.ts
import { Test, TestingModule } from '@nestjs/testing';
import { UserController } from './user.controller';
import { UserService } from './user.service';
import { CreateUserDto } from './dto/create-user.dto';
import { NotFoundException } from '@nestjs/common';

describe('UserController', () => {
  let controller: UserController;
  let service: UserService;

  const mockUserService = {
    createUser: jest.fn(),
    getUserById: jest.fn(),
    listUsers: jest.fn(),
    updateUser: jest.fn(),
    deleteUser: jest.fn(),
  };

  beforeEach(async () => {
    const module: TestingModule = await Test.createTestingModule({
      controllers: [UserController],
      providers: [
        {
          provide: UserService,
          useValue: mockUserService,
        },
      ],
    }).compile();

    controller = module.get<UserController>(UserController);
    service = module.get<UserService>(UserService);
    jest.clearAllMocks();
  });

  describe('POST /users', () => {
    it('should create user', async () => {
      const dto: CreateUserDto = {
        email: 'test@test.com',
        name: 'John',
        age: 30,
      };
      const created = { id: '1', ...dto };

      // Arrange
      mockUserService.createUser.mockResolvedValue(created);

      // Act
      const result = await controller.createUser(dto);

      // Assert
      expect(result).toEqual(created);
      expect(mockUserService.createUser).toHaveBeenCalledWith(dto);
    });
  });

  describe('GET /users/:id', () => {
    it('should return user', async () => {
      const user = { id: '1', email: 'test@test.com', name: 'John' };

      // Arrange
      mockUserService.getUserById.mockResolvedValue(user);

      // Act
      const result = await controller.getUser('1');

      // Assert
      expect(result).toEqual(user);
      expect(mockUserService.getUserById).toHaveBeenCalledWith('1');
    });

    it('should throw NotFoundException', async () => {
      // Arrange
      mockUserService.getUserById.mockRejectedValue(
        new NotFoundException('User not found')
      );

      // Act & Assert
      await expect(controller.getUser('999'))
        .rejects.toThrow(NotFoundException);
    });
  });
});
```

## Mocking Patterns

### Mock Repository Dependencies

```typescript
// Service depends on repository
@Injectable()
export class UserService {
  constructor(private readonly userRepository: UserRepository) {}
  // ...
}

// Mock in test
const mockUserRepository = {
  findById: jest.fn(),
  save: jest.fn(),
  update: jest.fn(),
  delete: jest.fn(),
};

const module = await Test.createTestingModule({
  providers: [
    UserService,
    {
      provide: UserRepository,
      useValue: mockUserRepository,
    },
  ],
}).compile();
```

### Mock External Services

```typescript
// Service depends on external service
@Injectable()
export class UserService {
  constructor(
    private readonly userRepository: UserRepository,
    private readonly emailService: EmailService,
  ) {}
  // ...
}

// Mock external service
const mockEmailService = {
  sendWelcome: jest.fn().mockResolvedValue(undefined),
  sendReset: jest.fn().mockResolvedValue(undefined),
};

const module = await Test.createTestingModule({
  providers: [
    UserService,
    {
      provide: UserRepository,
      useValue: mockUserRepository,
    },
    {
      provide: EmailService,
      useValue: mockEmailService,
    },
  ],
}).compile();
```

### Mock Third-Party Modules

```typescript
// Service uses third-party library
@Injectable()
export class PaymentService {
  constructor(
    private readonly stripe: StripeClient,
  ) {}
  // ...
}

// Mock third-party client
const mockStripe = {
  charges: {
    create: jest.fn(),
  },
};

const module = await Test.createTestingModule({
  providers: [
    PaymentService,
    {
      provide: StripeClient,
      useValue: mockStripe,
    },
  ],
}).compile();
```

## Test Organization

### Arrange-Act-Assert Pattern

```typescript
it('should create user with valid data', async () => {
  // Arrange - Set up test data and mocks
  const dto = { email: 'test@test.com', name: 'John', age: 30 };
  const expected = { id: '1', ...dto };
  mockUserRepository.save.mockResolvedValue(expected);

  // Act - Execute the code being tested
  const result = await service.createUser(dto);

  // Assert - Verify the results
  expect(result).toEqual(expected);
  expect(mockUserRepository.save).toHaveBeenCalled();
});
```

### Test One Thing Per Test

```typescript
// ❌ BAD - Multiple concerns
it('should create and email user', async () => {
  const dto = { email: 'test@test.com', name: 'John', age: 30 };
  mockUserRepository.save.mockResolvedValue({ id: '1', ...dto });
  mockEmailService.sendWelcome.mockResolvedValue(undefined);

  const result = await service.createUser(dto);

  expect(result.id).toBe('1');
  expect(mockUserRepository.save).toHaveBeenCalled();
  expect(mockEmailService.sendWelcome).toHaveBeenCalled();
});

// ✅ GOOD - One concern per test
it('should create user', async () => {
  const dto = { email: 'test@test.com', name: 'John', age: 30 };
  mockUserRepository.save.mockResolvedValue({ id: '1', ...dto });

  const result = await service.createUser(dto);

  expect(result.id).toBe('1');
});

it('should send welcome email after creating user', async () => {
  const dto = { email: 'test@test.com', name: 'John', age: 30 };
  mockUserRepository.save.mockResolvedValue({ id: '1', ...dto });
  mockEmailService.sendWelcome.mockResolvedValue(undefined);

  await service.createUser(dto);

  expect(mockEmailService.sendWelcome).toHaveBeenCalledWith('test@test.com');
});
```

### Describe Block Organization

```typescript
describe('UserService', () => {
  // Module setup
  let service: UserService;
  let repository: UserRepository;

  beforeEach(async () => {
    // ...
  });

  // Group related tests
  describe('createUser', () => {
    it('should create valid user', async () => { /* ... */ });
    it('should reject duplicate email', async () => { /* ... */ });
    it('should reject invalid age', async () => { /* ... */ });
  });

  describe('getUserById', () => {
    it('should return user if found', async () => { /* ... */ });
    it('should throw NotFoundException if not found', async () => { /* ... */ });
  });

  describe('updateUser', () => {
    it('should update user properties', async () => { /* ... */ });
    it('should not update email to duplicate', async () => { /* ... */ });
  });
});
```

## Edge Cases & Error Handling

### Test Error Paths

```typescript
describe('UserService', () => {
  describe('error handling', () => {
    it('should handle repository errors', async () => {
      // Arrange
      mockUserRepository.findById.mockRejectedValue(
        new Error('Database connection failed')
      );

      // Act & Assert
      await expect(service.getUserById('1'))
        .rejects.toThrow('Database connection failed');
    });

    it('should handle external service failures gracefully', async () => {
      const dto = { email: 'test@test.com', name: 'John', age: 30 };
      mockUserRepository.save.mockResolvedValue({ id: '1', ...dto });
      mockEmailService.sendWelcome.mockRejectedValue(
        new Error('Email service down')
      );

      // Should create user even if email fails
      const result = await service.createUser(dto);

      expect(result.id).toBe('1');
      expect(mockEmailService.sendWelcome).toHaveBeenCalled();
    });
  });
});
```

### Test Boundary Conditions

```typescript
describe('UserService', () => {
  describe('boundary conditions', () => {
    it('should handle minimum age', async () => {
      const dto = { email: 'test@test.com', name: 'John', age: 0 };
      mockUserRepository.findByEmail.mockResolvedValue(null);
      mockUserRepository.save.mockResolvedValue({ id: '1', ...dto });

      const result = await service.createUser(dto);

      expect(result.age).toBe(0);
    });

    it('should handle maximum age', async () => {
      const dto = { email: 'test@test.com', name: 'John', age: 150 };
      mockUserRepository.findByEmail.mockResolvedValue(null);

      await expect(service.createUser(dto))
        .rejects.toThrow(BadRequestException);
    });

    it('should handle empty strings', async () => {
      const dto = { email: '', name: '', age: 30 };

      // DTO validation should catch this
      // But service should also handle gracefully
      mockUserRepository.findByEmail.mockResolvedValue(null);

      // Service should reject empty email
      await expect(service.createUser(dto))
        .rejects.toThrow();
    });
  });
});
```

## Coverage Checklist

When writing tests, ensure:

- [ ] Happy path covered (normal usage)
- [ ] Error paths covered (exceptions thrown)
- [ ] Boundary conditions covered (min/max values)
- [ ] All mocked dependencies are called correctly
- [ ] All branches in code are executed
- [ ] All conditional logic is tested (if/else paths)
- [ ] All loops have iteration tests
- [ ] All error types are tested separately
- [ ] Service calls with mocks produce expected results
- [ ] External service failures are handled

## Generation Workflow

When generating tests for a new service:

1. **Create test file**: `<service-name>.spec.ts` in same folder
2. **Set up Test module**: Import service and all dependencies
3. **Mock all dependencies**: Create mock objects for repositories and services
4. **Write setup blocks**: `beforeEach` for module creation and mock clearing
5. **Group by method**: One `describe` block per method
6. **Test happy path**: Basic success case first
7. **Test error paths**: Each error condition separately
8. **Test edge cases**: Boundary values, empty inputs, etc.
9. **Verify mock calls**: Assert mocks were called with correct args
10. **Aim for 80%+ coverage**: Use `npm test -- --coverage` to check

## Code Review Checklist

When reviewing tests, verify:

- [ ] All dependencies are mocked
- [ ] No actual HTTP calls or database queries
- [ ] Mocks are reset between tests
- [ ] Tests are isolated (no interdependencies)
- [ ] One assertion focus per test
- [ ] Descriptive test names explain what's being tested
- [ ] Error cases tested as thoroughly as success cases
- [ ] Coverage is at or above minimum thresholds
- [ ] No skipped tests (`xit`, `skip`)
- [ ] No hardcoded delays or timeouts

## Additional Resources

- For mock patterns and advanced setup, see [mocking.md](mocking.md)
- For coverage configuration, see [coverage.md](coverage.md)
- For testing complex scenarios, see [scenarios.md](scenarios.md)
