# Database Strategies for E2E Testing

This document covers different approaches to managing test databases.

## Strategy Comparison

| Strategy | Speed | Cost | Isolation | Setup | Teardown |
|----------|-------|------|-----------|-------|----------|
| **In-Memory (SQLite)** | Very Fast | 0 | Per test | Easy | Automatic |
| **Separate DB Instance** | Fast | Low | Per suite | Medium | Manual |
| **Transaction Rollback** | Very Fast | 0 | Per test | Medium | Automatic |
| **Docker Compose** | Medium | Low | Per suite | Complex | Automatic |

## Strategy 1: In-Memory SQLite

**Best for:** Unit-like E2E tests, fast feedback, CI/CD

### Configuration

```typescript
// test/config/database.ts
export function getTestDatabase() {
  return {
    type: 'sqlite',
    database: ':memory:',
    entities: ['src/**/*.entity.ts'],
    synchronize: true,
    logging: false,
  };
}
```

### Test Setup

```typescript
describe('Users API - In-Memory', () => {
  let app: INestApplication;

  beforeAll(async () => {
    const module = await Test.createTestingModule({
      imports: [AppModule],
    })
      .overrideProvider('DATABASE_CONNECTION')
      .useValue(getTestDatabase())
      .compile();

    app = module.createNestApplication();
    await app.init();
  });

  afterAll(async () => {
    await app.close();
  });

  it('should create user', async () => {
    await request(app.getHttpServer())
      .post('/users')
      .send({ email: 'test@test.com', name: 'John', age: 30 })
      .expect(201);
  });
});
```

### Pros & Cons

✅ **Pros:**
- Extremely fast
- No external service needed
- Automatic cleanup
- Great for CI/CD

❌ **Cons:**
- SQLite quirks (different SQL dialect)
- Can't test database-specific features
- May not catch production bugs

## Strategy 2: Separate Test Database Instance

**Best for:** Production-like testing, full feature testing, local development

### PostgreSQL Setup

```bash
# Start test PostgreSQL
docker run -d \
  --name test-postgres \
  -e POSTGRES_USER=test_user \
  -e POSTGRES_PASSWORD=test_password \
  -e POSTGRES_DB=test_db \
  -p 5433:5432 \
  postgres:15-alpine
```

### Configuration

```typescript
// test/config/database.ts
export function getTestDatabase() {
  return {
    type: 'postgres',
    host: process.env.TEST_DB_HOST || 'localhost',
    port: parseInt(process.env.TEST_DB_PORT || '5433'),
    username: process.env.TEST_DB_USER || 'test_user',
    password: process.env.TEST_DB_PASSWORD || 'test_password',
    database: process.env.TEST_DB_NAME || 'test_db',
    entities: ['src/**/*.entity.ts'],
    synchronize: true,
    dropSchema: true,  // Clean schema
  };
}
```

### Test Cleanup

```typescript
class DatabaseHelper {
  static async clearAllTables(dataSource: DataSource): Promise<void> {
    const entities = dataSource.entityMetadatas;

    // Disable foreign keys
    await dataSource.query('SET FOREIGN_KEY_CHECKS = 0;');

    for (const entity of entities) {
      const repository = dataSource.getRepository(entity.name);
      await repository.delete({});
    }

    // Re-enable foreign keys
    await dataSource.query('SET FOREIGN_KEY_CHECKS = 1;');
  }

  static async resetSequences(dataSource: DataSource): Promise<void> {
    const entities = dataSource.entityMetadatas;

    for (const entity of entities) {
      const sequenceName = `${entity.tableName}_id_seq`;
      await dataSource.query(
        `ALTER SEQUENCE ${sequenceName} RESTART WITH 1;`
      ).catch(() => {
        // Sequence may not exist
      });
    }
  }
}

describe('Users API', () => {
  beforeEach(async () => {
    await DatabaseHelper.clearAllTables(dataSource);
    await DatabaseHelper.resetSequences(dataSource);
  });

  it('test 1', async () => { /* ... */ });
  it('test 2', async () => { /* ... */ });
});
```

### Pros & Cons

✅ **Pros:**
- Production-like environment
- Tests all database features
- Catches SQL dialect issues
- Can use database-specific features

❌ **Cons:**
- Slower than in-memory
- Requires external service
- Manual cleanup needed
- May fail in strict CI environments

## Strategy 3: Transaction Rollback (Fastest Clean State)

**Best for:** Large test suites, parallel execution, fast feedback

### Implementation

```typescript
// test/config/transaction.helper.ts
export class TransactionHelper {
  private queryRunners = new Map<string, QueryRunner>();

  async startTransaction(
    testName: string,
    dataSource: DataSource,
  ): Promise<void> {
    const queryRunner = dataSource.createQueryRunner();
    await queryRunner.startTransaction();
    this.queryRunners.set(testName, queryRunner);
  }

  async rollback(testName: string): Promise<void> {
    const queryRunner = this.queryRunners.get(testName);
    if (queryRunner) {
      await queryRunner.rollbackTransaction();
      await queryRunner.release();
      this.queryRunners.delete(testName);
    }
  }
}

const transactionHelper = new TransactionHelper();

describe('Users API - Transaction Rollback', () => {
  let app: INestApplication;
  let dataSource: DataSource;

  beforeAll(async () => {
    const module = await Test.createTestingModule({
      imports: [AppModule],
    }).compile();

    app = module.createNestApplication();
    await app.init();
    dataSource = module.get(DataSource);
  });

  beforeEach(async () => {
    await transactionHelper.startTransaction('current-test', dataSource);
  });

  afterEach(async () => {
    await transactionHelper.rollback('current-test');
  });

  afterAll(async () => {
    await app.close();
  });

  it('should create user', async () => {
    // Test runs in transaction
    const res = await request(app.getHttpServer())
      .post('/users')
      .send({ email: 'test@test.com', name: 'John', age: 30 })
      .expect(201);

    expect(res.body.id).toBeDefined();

    // User exists in transaction
    const user = await userRepository.findById(res.body.id);
    expect(user).toBeDefined();

    // Transaction auto-rolls back after test
  });

  it('should create another user independently', async () => {
    // Previous user doesn't exist (rolled back)
    const users = await userRepository.find();
    expect(users).toHaveLength(0);

    // Can create new user without conflicts
    await request(app.getHttpServer())
      .post('/users')
      .send({ email: 'test@test.com', name: 'John', age: 30 })
      .expect(201);
  });
});
```

### Pros & Cons

✅ **Pros:**
- Extremely fast cleanup
- Database unchanged after test
- Good for parallel execution
- Tests run in isolation

❌ **Cons:**
- Some ORM features may not work in transactions
- Can't test transaction scenarios itself
- Complex setup

## Strategy 4: Docker Compose (Full Stack)

**Best for:** Full integration testing, CI/CD, microservices

### docker-compose.test.yml

```yaml
version: '3.8'

services:
  postgres:
    image: postgres:15-alpine
    environment:
      POSTGRES_USER: test_user
      POSTGRES_PASSWORD: test_password
      POSTGRES_DB: test_db
    ports:
      - '5433:5432'
    healthcheck:
      test: ['CMD-SHELL', 'pg_isready -U test_user']
      interval: 10s
      timeout: 5s
      retries: 5
    volumes:
      - test_postgres_data:/var/lib/postgresql/data

  redis:
    image: redis:7-alpine
    ports:
      - '6380:6379'
    healthcheck:
      test: ['CMD', 'redis-cli', 'ping']
      interval: 10s
      timeout: 5s
      retries: 5

volumes:
  test_postgres_data:
```

### Test Setup

```typescript
describe('Full Stack E2E', () => {
  let app: INestApplication;

  beforeAll(async () => {
    // Ensure services are ready
    await waitForHealthcheck('postgres', 'localhost:5433', 30000);
    await waitForHealthcheck('redis', 'localhost:6380', 30000);

    const module = await Test.createTestingModule({
      imports: [AppModule],
    }).compile();

    app = module.createNestApplication();
    await app.init();
  });

  afterAll(async () => {
    await app.close();
  });

  it('should create and cache user', async () => {
    // Creates in PostgreSQL
    const res = await request(app.getHttpServer())
      .post('/users')
      .send({ email: 'test@test.com', name: 'John', age: 30 })
      .expect(201);

    // Caches in Redis
    await new Promise(resolve => setTimeout(resolve, 100));

    // Second request hits cache
    const cached = await request(app.getHttpServer())
      .get(`/users/${res.body.id}`)
      .expect(200);

    expect(cached.body).toEqual(res.body);
  });
});
```

### npm Script

```json
{
  "scripts": {
    "test:e2e:full": "npm run test:e2e:services:up && npm run test:e2e && npm run test:e2e:services:down",
    "test:e2e:services:up": "docker-compose -f docker-compose.test.yml up -d",
    "test:e2e:services:down": "docker-compose -f docker-compose.test.yml down",
    "test:e2e": "jest --config test/jest-e2e.json"
  }
}
```

### Pros & Cons

✅ **Pros:**
- Full production-like environment
- Tests all integrations
- Repeatable across machines
- Good documentation

❌ **Cons:**
- Slowest setup/teardown
- Requires Docker
- More complex configuration
- Resource intensive

## Hybrid Approach (Recommended)

**Use different strategies based on context:**

### Local Development

Use **separate PostgreSQL instance** for production-like testing:

```bash
# .env.local
TEST_DB_HOST=localhost
TEST_DB_PORT=5433
TEST_DB_NAME=test_db_local
```

### CI/CD

Use **Docker Compose** for isolated, reproducible environment:

```yaml
# GitHub Actions
services:
  postgres:
    image: postgres:15-alpine
    options: >-
      --health-cmd pg_isready
```

### Quick Feedback Loop

Use **in-memory SQLite** for fast unit-like tests:

```json
{
  "scripts": {
    "test:e2e:fast": "jest --config test/jest-e2e-sqlite.json",
    "test:e2e:full": "jest --config test/jest-e2e-postgres.json"
  }
}
```

### Configuration

```typescript
// test/config/database.ts
export function getTestDatabase() {
  const strategy = process.env.TEST_STRATEGY || 'sqlite';

  switch (strategy) {
    case 'postgres':
      return postgresConfig();
    case 'transaction':
      return postgresConfig(); // Used with transaction helper
    case 'sqlite':
    default:
      return sqliteConfig();
  }
}
```

**Run with different strategies:**

```bash
# Fast feedback
TEST_STRATEGY=sqlite npm run test:e2e

# Production-like
TEST_STRATEGY=postgres npm run test:e2e

# With transaction rollback
TEST_STRATEGY=transaction npm run test:e2e
```

## Database State Management Patterns

### Pattern 1: Minimal Setup (Test-Specific Data)

```typescript
it('should return user by email', async () => {
  // Create only what's needed
  const user = await userRepository.save({
    email: 'test@test.com',
    name: 'John',
    age: 30,
  });

  await request(app.getHttpServer())
    .get('/users/by-email/test@test.com')
    .expect(200)
    .expect(res => {
      expect(res.body.id).toBe(user.id);
    });
});
```

### Pattern 2: Seed and Clear

```typescript
beforeEach(async () => {
  // Clear
  await userRepository.clear();

  // Seed
  await userRepository.saveMany([
    { email: 'user1@test.com', name: 'User 1' },
    { email: 'user2@test.com', name: 'User 2' },
  ]);
});

it('should list users', async () => {
  await request(app.getHttpServer())
    .get('/users')
    .expect(200)
    .expect(res => {
      expect(res.body).toHaveLength(2);
    });
});
```

### Pattern 3: Factory Pattern

```typescript
class UserFactory {
  static async create(repository: UserRepository, overrides = {}): Promise<User> {
    return repository.save({
      email: 'test@test.com',
      name: 'Test User',
      age: 30,
      ...overrides,
    });
  }

  static async createMany(
    repository: UserRepository,
    count: number,
  ): Promise<User[]> {
    return Promise.all(
      Array.from({ length: count }).map(() => this.create(repository))
    );
  }
}

it('should paginate users', async () => {
  await UserFactory.createMany(userRepository, 25);

  await request(app.getHttpServer())
    .get('/users?page=1&limit=10')
    .expect(200)
    .expect(res => {
      expect(res.body).toHaveLength(10);
    });
});
```
