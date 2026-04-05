# Test Environment Setup

This document covers configuring and managing test environments for E2E testing.

## Database Setup

### PostgreSQL Test Database

**Install Docker (recommended for isolation):**

```bash
docker pull postgres:15-alpine
```

**Start test database:**

```bash
docker run -d \
  --name postgres-test \
  -e POSTGRES_USER=test_user \
  -e POSTGRES_PASSWORD=test_password \
  -e POSTGRES_DB=test_db \
  -p 5433:5432 \
  postgres:15-alpine
```

**Stop test database:**

```bash
docker stop postgres-test
docker rm postgres-test
```

### In-Memory Database (SQLite)

For faster tests, use SQLite in-memory:

```typescript
// test/config/database.ts
export function getTestDatabase() {
  return {
    type: 'sqlite',
    database: ':memory:', // In-memory
    entities: ['src/**/*.entity.ts'],
    synchronize: true,
    logging: false,
  };
}
```

**Pros:**
- Very fast
- No external service needed
- Automatic cleanup

**Cons:**
- Different SQL dialect (may not match production)
- Limited features vs PostgreSQL

### TypeORM Test Configuration

```typescript
// test/config/typeorm.config.ts
import { TypeOrmModuleOptions } from '@nestjs/typeorm';
import * as entities from '../../src/**/*.entity';

export const testTypeOrmConfig: TypeOrmModuleOptions = {
  type: 'postgres',
  host: process.env.TEST_DB_HOST || 'localhost',
  port: parseInt(process.env.TEST_DB_PORT || '5433'),
  username: process.env.TEST_DB_USER || 'test_user',
  password: process.env.TEST_DB_PASSWORD || 'test_password',
  database: process.env.TEST_DB_NAME || 'test_db',
  entities: ['src/**/*.entity.ts'],
  
  // Test-specific settings
  synchronize: true,        // Auto-create schema
  dropSchema: true,         // Clean before each run
  logging: process.env.TEST_DB_VERBOSE === 'true',
  
  // Performance
  maxQueryExecutionTime: 5000,
  extra: {
    poolSize: 5,
  },
};
```

### Prisma Test Configuration

```typescript
// test/config/prisma.service.ts
import { PrismaClient } from '@prisma/client';

export function getPrismaTestClient() {
  const prisma = new PrismaClient({
    datasources: {
      db: {
        url: process.env.TEST_DATABASE_URL || 
             'postgresql://test_user:test_password@localhost:5433/test_db',
      },
    },
  });

  return prisma;
}
```

## Environment Variables

### .env.test

```bash
# Database
TEST_DB_HOST=localhost
TEST_DB_PORT=5433
TEST_DB_USER=test_user
TEST_DB_PASSWORD=test_password
TEST_DB_NAME=test_db
TEST_DB_VERBOSE=false

# App
NODE_ENV=test
APP_PORT=3001
LOG_LEVEL=error

# External services (use test/mock endpoints)
STRIPE_API_KEY=sk_test_123456
SENDGRID_API_KEY=mock_key
JWT_SECRET=test_secret_key_only_for_testing

# Timeouts
TEST_TIMEOUT=30000
DATABASE_TIMEOUT=10000
```

### Loading Test Environment

```typescript
// test/setup.ts
import * as dotenv from 'dotenv';
import * as path from 'path';

// Load .env.test before any other code
dotenv.config({ path: path.join(__dirname, '../.env.test') });
```

**Configure Jest to run setup:**

```json
{
  "jest": {
    "setupFilesAfterEnv": ["<rootDir>/test/setup.ts"]
  }
}
```

## Database Seeding

### Seed Helper

```typescript
// test/seeds/user.seed.ts
import { UserRepository } from '../../src/users/user.repository';
import { User } from '../../src/users/entities/user.entity';

export class UserSeed {
  static async create(repository: UserRepository, count: number = 10): Promise<User[]> {
    const users = [];

    for (let i = 0; i < count; i++) {
      const user = new User();
      user.email = `user${i}@test.com`;
      user.name = `Test User ${i}`;
      user.age = 20 + i;
      users.push(user);
    }

    return repository.saveMany(users);
  }

  static async createWithEmail(
    repository: UserRepository,
    email: string,
  ): Promise<User> {
    const user = new User();
    user.email = email;
    user.name = email.split('@')[0];
    user.age = 30;

    return repository.save(user);
  }
}

// test/seeds/product.seed.ts
export class ProductSeed {
  static async create(
    repository: ProductRepository,
    count: number = 5,
  ): Promise<Product[]> {
    const products = [];

    for (let i = 0; i < count; i++) {
      const product = new Product();
      product.name = `Product ${i}`;
      product.price = 99.99 + i;
      product.stock = 100;
      products.push(product);
    }

    return repository.saveMany(products);
  }
}
```

### Usage in Tests

```typescript
describe('Orders API', () => {
  let productRepository: ProductRepository;
  let userRepository: UserRepository;

  beforeEach(async () => {
    // Clear all data
    await productRepository.clear();
    await userRepository.clear();

    // Seed test data
    await ProductSeed.create(productRepository, 5);
    await UserSeed.create(userRepository, 3);
  });

  it('should list all products', async () => {
    await request(app.getHttpServer())
      .get('/products')
      .expect(200)
      .expect(res => {
        expect(res.body).toHaveLength(5);
      });
  });
});
```

## Test Database Cleanup Strategies

### Strategy 1: Truncate All Tables

```typescript
class DatabaseHelper {
  static async truncateAllTables(dataSource: DataSource): Promise<void> {
    const entities = dataSource.entityMetadatas;

    // Disable foreign key checks
    await dataSource.query('SET FOREIGN_KEY_CHECKS = 0;');

    for (const entity of entities) {
      const repository = dataSource.getRepository(entity.name);
      await repository.query(`TRUNCATE TABLE ${entity.tableName};`);
    }

    // Re-enable foreign key checks
    await dataSource.query('SET FOREIGN_KEY_CHECKS = 1;');
  }
}
```

### Strategy 2: Transaction Rollback

```typescript
describe('Users API', () => {
  let queryRunner: QueryRunner;

  beforeEach(async () => {
    // Start transaction
    queryRunner = dataSource.createQueryRunner();
    await queryRunner.startTransaction();
  });

  afterEach(async () => {
    // Automatic rollback
    await queryRunner.rollbackTransaction();
    await queryRunner.release();
  });

  it('should create user', async () => {
    // Changes are in transaction, auto-rolled back
    await request(app.getHttpServer())
      .post('/users')
      .send(userData)
      .expect(201);
  });
});
```

### Strategy 3: Snapshots

```typescript
class DatabaseSnapshot {
  static async take(dataSource: DataSource): Promise<Map<string, any[]>> {
    const snapshot = new Map();
    const entities = dataSource.entityMetadatas;

    for (const entity of entities) {
      const data = await dataSource.query(`SELECT * FROM ${entity.tableName};`);
      snapshot.set(entity.name, data);
    }

    return snapshot;
  }

  static async restore(
    dataSource: DataSource,
    snapshot: Map<string, any[]>,
  ): Promise<void> {
    // Clear all tables
    for (const [tableName] of snapshot) {
      await dataSource.query(`TRUNCATE TABLE ${tableName} CASCADE;`);
    }

    // Restore data
    for (const [tableName, data] of snapshot) {
      for (const row of data) {
        await dataSource.getRepository(tableName).save(row);
      }
    }
  }
}
```

## Performance Optimization

### Connection Pooling

```typescript
{
  type: 'postgres',
  host: 'localhost',
  port: 5433,
  username: 'test_user',
  password: 'test_password',
  database: 'test_db',
  extra: {
    max: 5,                    // Max connections
    idleTimeoutMillis: 30000,
    connectionTimeoutMillis: 2000,
  },
}
```

### Parallel Test Execution

Run tests in parallel (if database supports):

```json
{
  "jest": {
    "maxWorkers": 4,
    "testTimeout": 30000
  }
}
```

**Or run sequential to avoid conflicts:**

```json
{
  "jest": {
    "maxWorkers": 1,
    "testTimeout": 30000
  }
}
```

### Index Coverage

Ensure test database has same indexes as production:

```typescript
// test/migrations/001_create_indexes.ts
export async function up(dataSource: DataSource) {
  await dataSource.query(`
    CREATE INDEX idx_users_email ON users(email);
    CREATE INDEX idx_orders_user_id ON orders(user_id);
  `);
}
```

## CI/CD Environment Variables

### GitHub Actions

```yaml
# .github/workflows/e2e.yml
name: E2E Tests

on: [push, pull_request]

jobs:
  e2e:
    runs-on: ubuntu-latest
    
    services:
      postgres:
        image: postgres:15-alpine
        env:
          POSTGRES_USER: test_user
          POSTGRES_PASSWORD: test_password
          POSTGRES_DB: test_db
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 5432:5432

    steps:
      - uses: actions/checkout@v2
      - uses: actions/setup-node@v2
        with:
          node-version: '18'
      
      - run: npm install
      
      - name: Run E2E tests
        env:
          TEST_DB_HOST: postgres
          TEST_DB_PORT: 5432
          TEST_DB_USER: test_user
          TEST_DB_PASSWORD: test_password
          TEST_DB_NAME: test_db
        run: npm run test:e2e
```

## Local Development Setup

### Docker Compose for Full Stack

```yaml
# docker-compose.test.yml
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

  redis:
    image: redis:7-alpine
    ports:
      - '6380:6379'
    healthcheck:
      test: ['CMD', 'redis-cli', 'ping']
      interval: 10s
      timeout: 5s
      retries: 5
```

**Start for E2E testing:**

```bash
docker-compose -f docker-compose.test.yml up -d
npm run test:e2e
docker-compose -f docker-compose.test.yml down
```

## Debugging Tests

### Enable Database Logging

```typescript
beforeAll(async () => {
  const module = await Test.createTestingModule({
    imports: [AppModule],
  })
    .overrideProvider('DATABASE_CONNECTION')
    .useFactory(() => ({
      ...testConfig,
      logging: process.env.DEBUG_SQL === 'true', // Enable with DEBUG_SQL=true
    }))
    .compile();
});
```

**Run with SQL logging:**

```bash
DEBUG_SQL=true npm run test:e2e
```

### Print Test Data

```typescript
afterEach(async () => {
  if (process.env.DEBUG_STATE === 'true') {
    const users = await userRepository.find();
    console.log('Current users:', JSON.stringify(users, null, 2));
  }
});
```

**Run with state logging:**

```bash
DEBUG_STATE=true npm run test:e2e
```

### VSCode Debugging

```json
// .vscode/launch.json
{
  "type": "node",
  "request": "launch",
  "name": "E2E Tests",
  "program": "${workspaceFolder}/node_modules/.bin/jest",
  "args": ["--config", "test/jest-e2e.json", "--runInBand"],
  "console": "integratedTerminal",
  "internalConsoleOptions": "neverOpen"
}
```

## Performance Testing Checklist

- [ ] Test database is isolated and fast
- [ ] Connection pooling is configured
- [ ] Unnecessary logging is disabled in tests
- [ ] Database indexes match production
- [ ] Cleanup between tests is efficient
- [ ] Tests can run in parallel or serial as needed
- [ ] Timeouts are reasonable (not too strict)
