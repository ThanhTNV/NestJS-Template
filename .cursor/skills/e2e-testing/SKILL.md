---
name: e2e-testing
description: Write comprehensive E2E tests for NestJS APIs using Jest and SuperTest. Test full request/response cycles including database interactions, external services, and state changes. Enforce test isolation, reproducibility, and proper environment setup/teardown.
---

# E2E Testing

This skill guides writing E2E tests for NestJS applications that validate controllers, API endpoints, and full integration flows using test databases.

## What is E2E Testing?

E2E tests verify the full request/response cycle:

```
HTTP Request → Controller → Service → Repository → Database → Response
   ↑                                                              ↓
   └──────────────────── Assert ─────────────────────────────────┘
```

Unlike unit tests (mock dependencies), E2E tests use **real** (or test) databases and services.

## Quick Reference

| Aspect | Pattern | Location |
|--------|---------|----------|
| **Test file** | `<feature>.e2e-spec.ts` | `test/` |
| **Test DB** | Separate instance per test suite | `test/db/` |
| **Setup** | `beforeAll()` spin up, `afterAll()` teardown | Module-level |
| **Isolation** | Clear data between tests | `beforeEach()` / `afterEach()` |
| **HTTP client** | SuperTest (`request()`) | Built into test |

## Full Test Setup Template

### E2E Test File Structure

```typescript
// test/user.e2e-spec.ts
import { Test, TestingModule } from '@nestjs/testing';
import { INestApplication, ValidationPipe } from '@nestjs/common';
import * as request from 'supertest';
import { AppModule } from '../src/app.module';
import { UserRepository } from '../src/users/user.repository';

describe('User API (e2e)', () => {
  let app: INestApplication;
  let userRepository: UserRepository;

  // Setup: Create app and connect to test database
  beforeAll(async () => {
    const moduleFixture: TestingModule = await Test.createTestingModule({
      imports: [AppModule], // Import full app
    })
      .overrideProvider('DATABASE_CONNECTION')
      .useValue(getTestDatabase()) // Override with test DB
      .compile();

    app = moduleFixture.createNestApplication();
    app.useGlobalPipes(new ValidationPipe());
    await app.init();

    userRepository = moduleFixture.get<UserRepository>(UserRepository);
  });

  // Cleanup: Destroy app and close connections
  afterAll(async () => {
    await app.close();
  });

  // Isolation: Clear data before each test
  beforeEach(async () => {
    await userRepository.clear(); // Or: DELETE FROM users
  });

  describe('POST /users', () => {
    it('should create user with valid data', async () => {
      const createDto = {
        email: 'test@test.com',
        name: 'John',
        age: 30,
      };

      // Act: Make HTTP request
      const response = await request(app.getHttpServer())
        .post('/users')
        .send(createDto)
        .expect(201); // Assert status code

      // Assert: Verify response body
      expect(response.body).toEqual(
        expect.objectContaining({
          email: 'test@test.com',
          name: 'John',
          age: 30,
        })
      );
      expect(response.body.id).toBeDefined();

      // Assert: Verify database state
      const savedUser = await userRepository.findById(response.body.id);
      expect(savedUser).toBeDefined();
      expect(savedUser.email).toBe('test@test.com');
    });

    it('should reject duplicate email', async () => {
      // Setup: Create existing user
      await request(app.getHttpServer())
        .post('/users')
        .send({ email: 'test@test.com', name: 'John', age: 30 })
        .expect(201);

      // Act: Try to create duplicate
      const response = await request(app.getHttpServer())
        .post('/users')
        .send({ email: 'test@test.com', name: 'Jane', age: 25 })
        .expect(409); // Conflict

      // Assert: Error message
      expect(response.body.message).toContain('already in use');
    });

    it('should reject invalid data', async () => {
      await request(app.getHttpServer())
        .post('/users')
        .send({ email: 'invalid', name: '', age: 200 })
        .expect(400) // Bad Request
        .expect(res => {
          expect(res.body.message).toContain('validation');
        });
    });
  });

  describe('GET /users/:id', () => {
    it('should return user by ID', async () => {
      // Setup: Create user
      const createRes = await request(app.getHttpServer())
        .post('/users')
        .send({ email: 'test@test.com', name: 'John', age: 30 })
        .expect(201);

      const userId = createRes.body.id;

      // Act & Assert
      await request(app.getHttpServer())
        .get(`/users/${userId}`)
        .expect(200)
        .expect(res => {
          expect(res.body.id).toBe(userId);
          expect(res.body.email).toBe('test@test.com');
        });
    });

    it('should return 404 for non-existent user', async () => {
      await request(app.getHttpServer())
        .get('/users/999')
        .expect(404)
        .expect(res => {
          expect(res.body.message).toContain('not found');
        });
    });
  });

  describe('PUT /users/:id', () => {
    it('should update user', async () => {
      // Setup
      const createRes = await request(app.getHttpServer())
        .post('/users')
        .send({ email: 'test@test.com', name: 'John', age: 30 })
        .expect(201);

      const userId = createRes.body.id;

      // Act
      await request(app.getHttpServer())
        .put(`/users/${userId}`)
        .send({ name: 'Jane', age: 28 })
        .expect(200)
        .expect(res => {
          expect(res.body.name).toBe('Jane');
          expect(res.body.age).toBe(28);
          expect(res.body.email).toBe('test@test.com'); // Unchanged
        });

      // Assert: Database
      const updated = await userRepository.findById(userId);
      expect(updated.name).toBe('Jane');
    });
  });

  describe('DELETE /users/:id', () => {
    it('should delete user', async () => {
      // Setup
      const createRes = await request(app.getHttpServer())
        .post('/users')
        .send({ email: 'test@test.com', name: 'John', age: 30 })
        .expect(201);

      const userId = createRes.body.id;

      // Act
      await request(app.getHttpServer())
        .delete(`/users/${userId}`)
        .expect(204); // No content

      // Assert: Database
      const deleted = await userRepository.findById(userId);
      expect(deleted).toBeNull();
    });
  });
});
```

## SuperTest HTTP Methods

```typescript
// GET request
request(app.getHttpServer())
  .get('/users')
  .query({ limit: 10 })
  .set('Authorization', 'Bearer token')
  .expect(200)

// POST request
request(app.getHttpServer())
  .post('/users')
  .send({ email: 'test@test.com', name: 'John' })
  .set('Content-Type', 'application/json')
  .expect(201)

// PUT request
request(app.getHttpServer())
  .put('/users/1')
  .send({ name: 'Updated' })
  .expect(200)

// DELETE request
request(app.getHttpServer())
  .delete('/users/1')
  .expect(204)

// With authentication
request(app.getHttpServer())
  .post('/orders')
  .set('Authorization', 'Bearer eyJhbGc...')
  .send(orderData)
  .expect(201)
```

## Test Isolation Patterns

### Clear Database Between Tests

```typescript
describe('Users API', () => {
  let userRepository: UserRepository;

  beforeEach(async () => {
    // Clear all users before each test
    await userRepository.clear();
  });

  it('test 1', async () => {
    // Can safely assume empty database
    await request(app.getHttpServer()).post('/users').send(data).expect(201);
  });

  it('test 2', async () => {
    // Database is clean again
    await request(app.getHttpServer()).get('/users').expect(200).expect(res => {
      expect(res.body).toHaveLength(0); // Always empty
    });
  });
});
```

### Transaction Isolation

```typescript
describe('Orders API', () => {
  beforeEach(async () => {
    // Start transaction before each test
    await database.startTransaction();
  });

  afterEach(async () => {
    // Rollback after each test (automatic cleanup)
    await database.rollback();
  });

  it('should create order', async () => {
    // Changes are automatically rolled back
  });
});
```

### Cleanup Helpers

```typescript
class TestHelper {
  static async clearAllTables(dataSource: DataSource): Promise<void> {
    const entities = dataSource.entityMetadatas;

    for (const entity of entities) {
      const repository = dataSource.getRepository(entity.name);
      await repository.clear();
    }
  }

  static async seedUsers(userRepository: UserRepository): Promise<User[]> {
    return userRepository.saveMany([
      { email: 'user1@test.com', name: 'User 1', age: 25 },
      { email: 'user2@test.com', name: 'User 2', age: 30 },
      { email: 'user3@test.com', name: 'User 3', age: 35 },
    ]);
  }
}

describe('Users API', () => {
  beforeEach(async () => {
    await TestHelper.clearAllTables(dataSource);
  });

  it('should list all users', async () => {
    await TestHelper.seedUsers(userRepository);

    await request(app.getHttpServer())
      .get('/users')
      .expect(200)
      .expect(res => {
        expect(res.body).toHaveLength(3);
      });
  });
});
```

## Complex Flow Testing

### Multi-Step User Journeys

```typescript
describe('Complete User Workflow', () => {
  it('should register, login, and purchase', async () => {
    // Step 1: Register
    const registerRes = await request(app.getHttpServer())
      .post('/auth/register')
      .send({
        email: 'user@test.com',
        password: 'Password123!',
        name: 'John',
      })
      .expect(201);

    const userId = registerRes.body.id;

    // Step 2: Login
    const loginRes = await request(app.getHttpServer())
      .post('/auth/login')
      .send({
        email: 'user@test.com',
        password: 'Password123!',
      })
      .expect(200);

    const token = loginRes.body.accessToken;

    // Step 3: Create order (authenticated)
    const orderRes = await request(app.getHttpServer())
      .post('/orders')
      .set('Authorization', `Bearer ${token}`)
      .send({
        productId: '1',
        quantity: 2,
      })
      .expect(201);

    expect(orderRes.body.userId).toBe(userId);

    // Step 4: Verify order in database
    const order = await orderRepository.findById(orderRes.body.id);
    expect(order.userId).toBe(userId);
  });
});
```

### External Service Mocking in E2E

```typescript
describe('Payment API', () => {
  beforeAll(async () => {
    // Mock Stripe for E2E tests
    jest.mock('stripe', () => ({
      Stripe: jest.fn().mockImplementation(() => ({
        charges: {
          create: jest.fn().mockResolvedValue({
            id: 'ch_test_123',
            amount: 1000,
            status: 'succeeded',
          }),
        },
      })),
    }));

    // ...rest of setup
  });

  it('should process payment', async () => {
    await request(app.getHttpServer())
      .post('/payments')
      .send({
        amount: 100,
        currency: 'USD',
        description: 'Test payment',
      })
      .expect(200)
      .expect(res => {
        expect(res.body.status).toBe('succeeded');
      });
  });
});
```

## Error Testing

### Status Code Verification

```typescript
describe('Error Handling', () => {
  it('should return 400 for invalid input', async () => {
    await request(app.getHttpServer())
      .post('/users')
      .send({ email: 'invalid', age: 'not-a-number' })
      .expect(400);
  });

  it('should return 401 for unauthorized access', async () => {
    await request(app.getHttpServer())
      .post('/orders')
      .send(orderData)
      .expect(401);
  });

  it('should return 403 for forbidden resource', async () => {
    await request(app.getHttpServer())
      .put('/users/other-user-id')
      .set('Authorization', `Bearer ${myToken}`)
      .send({ name: 'Hacked' })
      .expect(403);
  });

  it('should return 404 for not found', async () => {
    await request(app.getHttpServer())
      .get('/users/999')
      .expect(404);
  });

  it('should return 409 for conflict', async () => {
    // Create user
    await request(app.getHttpServer())
      .post('/users')
      .send({ email: 'test@test.com', name: 'John', age: 30 })
      .expect(201);

    // Try to create duplicate
    await request(app.getHttpServer())
      .post('/users')
      .send({ email: 'test@test.com', name: 'Jane', age: 25 })
      .expect(409);
  });
});
```

## Configuration

### package.json Setup

```json
{
  "scripts": {
    "test:e2e": "jest --config ./test/jest-e2e.json",
    "test:e2e:watch": "jest --config ./test/jest-e2e.json --watch",
    "test:e2e:cov": "jest --config ./test/jest-e2e.json --coverage"
  }
}
```

### jest-e2e.json

```json
{
  "moduleFileExtensions": ["js", "json", "ts"],
  "rootDir": ".",
  "testEnvironment": "node",
  "testRegex": ".e2e-spec.ts$",
  "transform": {
    "^.+\\.(t|j)s$": "ts-jest"
  },
  "collectCoverageFrom": [
    "../src/**/*.(t|j)s",
    "!../src/**/*.module.ts",
    "!../src/**/main.ts"
  ],
  "coverageDirectory": "../coverage-e2e",
  "testTimeout": 30000
}
```

## Test Database Strategy

### Separate Test Database

```typescript
// test/config/database.ts
export function getTestDatabase() {
  return {
    type: 'postgres',
    host: process.env.TEST_DB_HOST || 'localhost',
    port: parseInt(process.env.TEST_DB_PORT || '5432'),
    username: process.env.TEST_DB_USER || 'test_user',
    password: process.env.TEST_DB_PASSWORD || 'test_password',
    database: process.env.TEST_DB_NAME || 'test_db',
    entities: ['src/**/*.entity.ts'],
    synchronize: true, // Auto-sync schema for tests
    dropSchema: true,  // Clean schema before each run
  };
}
```

### Environment Variables

```bash
# .env.test
TEST_DB_HOST=localhost
TEST_DB_PORT=5432
TEST_DB_USER=test_user
TEST_DB_PASSWORD=test_password
TEST_DB_NAME=test_db_e2e
NODE_ENV=test
```

## Checklist for E2E Tests

When writing E2E tests, verify:

- [ ] App is initialized with full module hierarchy
- [ ] Database is clean before each test
- [ ] All HTTP status codes are tested (success + errors)
- [ ] Request/response bodies are validated
- [ ] Database state is verified after operations
- [ ] Multi-step workflows are tested end-to-end
- [ ] Authentication/authorization is tested
- [ ] External services are mocked appropriately
- [ ] Cleanup happens in `afterAll()` and `afterEach()`
- [ ] Tests are isolated (no interdependencies)
- [ ] Error messages are user-friendly

## Generation Workflow

When generating E2E tests for a new feature:

1. **Create test file**: `test/<feature>.e2e-spec.ts`
2. **Setup app**: Create NestApplication with full module
3. **Override providers**: Replace real DB with test DB
4. **Setup fixtures**: Get repositories and helper services
5. **Create cleanup**: `beforeEach()` clear, `afterAll()` close
6. **Test happy path**: Normal success scenarios first
7. **Test errors**: Each error code separately
8. **Test complex flows**: Multi-step user journeys
9. **Verify DB state**: Assert changes persisted
10. **Run test**: Execute to ensure isolation

## Review Checklist

When reviewing E2E tests:

- [ ] Tests start with clean state
- [ ] Tests don't interfere with each other
- [ ] Real database or equivalent is used
- [ ] HTTP requests use SuperTest correctly
- [ ] Response bodies and status codes verified
- [ ] Database state checked after changes
- [ ] Error scenarios explicitly tested
- [ ] Cleanup in `afterAll()` and `afterEach()`
- [ ] Tests timeout configured appropriately
- [ ] No hardcoded IDs or timestamps

## Additional Resources

- For environment setup details, see [environment.md](environment.md)
- For database strategies, see [database.md](database.md)
- For advanced testing patterns, see [advanced.md](advanced.md)
