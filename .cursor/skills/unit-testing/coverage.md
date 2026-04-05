# Coverage Configuration & Strategy

This document covers setting up and enforcing coverage thresholds in NestJS projects.

## Coverage Metrics

### Branches (Conditional Coverage)

Percentage of conditional branches executed (`if`, `else`, `ternary`, `switch`).

```typescript
function getDiscount(age: number): number {
  if (age < 18) return 0.1;      // Branch 1
  else if (age > 65) return 0.15; // Branch 2
  else return 0.05;               // Branch 3
}

// To achieve 100% branch coverage:
test('age < 18', () => expect(getDiscount(15)).toBe(0.1));
test('age > 65', () => expect(getDiscount(70)).toBe(0.15));
test('age 18-65', () => expect(getDiscount(30)).toBe(0.05));
```

### Functions (Function Coverage)

Percentage of functions called at least once.

```typescript
class UserService {
  createUser() { /* ... */ }
  getUser() { /* ... */ }
  updateUser() { /* ... */ }
}

// To achieve 100% function coverage:
test('createUser is called', async () => { /* ... */ });
test('getUser is called', async () => { /* ... */ });
test('updateUser is called', async () => { /* ... */ });
```

### Lines (Line Coverage)

Percentage of lines of code executed.

```typescript
function process(value: number) {
  const doubled = value * 2;  // Line 1
  const result = doubled + 1; // Line 2
  return result;              // Line 3
}

// All 3 lines must execute
test('should process value', () => {
  expect(process(5)).toBe(11);
});
```

### Statements (Statement Coverage)

Percentage of statements executed.

```typescript
class Repository {
  async save(data: any) {
    const id = generateId();        // Statement 1
    const record = { ...data, id }; // Statement 2
    await db.insert(record);        // Statement 3
    return record;                  // Statement 4
  }
}

// All 4 statements must execute
test('should save data', async () => {
  const result = await repository.save({ name: 'test' });
  expect(result.id).toBeDefined();
});
```

## Jest Configuration

### package.json Setup

```json
{
  "scripts": {
    "test": "jest",
    "test:watch": "jest --watch",
    "test:cov": "jest --coverage",
    "test:cov:strict": "jest --coverage --coverageThreshold=./jest.config.js"
  },
  "jest": {
    "moduleFileExtensions": ["js", "json", "ts"],
    "rootDir": "src",
    "testRegex": ".*\\.spec\\.ts$",
    "transform": {
      "^.+\\.(t|j)s$": "ts-jest"
    },
    "collectCoverageFrom": [
      "**/*.(t|j)s",
      "!**/*.module.ts",
      "!**/main.ts",
      "!**/*.entity.ts",
      "!**/dto/*"
    ],
    "coverageDirectory": "../coverage",
    "testEnvironment": "node"
  }
}
```

### jest.config.js with Thresholds

```javascript
module.exports = {
  moduleFileExtensions: ['js', 'json', 'ts'],
  rootDir: 'src',
  testRegex: '.*\\.spec\\.ts$',
  transform: {
    '^.+\\.(t|j)s$': 'ts-jest',
  },
  collectCoverageFrom: [
    '**/*.(t|j)s',
    '!**/*.module.ts',
    '!**/main.ts',
  ],
  coverageDirectory: '../coverage',
  testEnvironment: 'node',

  // Coverage thresholds
  coverageThreshold: {
    global: {
      branches: 75,
      functions: 80,
      lines: 80,
      statements: 80,
    },
    // Different thresholds for different file patterns
    './src/services/': {
      branches: 85,
      functions: 90,
      lines: 90,
      statements: 90,
    },
    './src/repositories/': {
      branches: 85,
      functions: 85,
      lines: 85,
      statements: 85,
    },
  },
};
```

## Coverage Commands

### Run Coverage Report

```bash
npm run test:cov
```

Output:
```
--------------|----------|----------|----------|----------|
File          | % Stmts  | % Branch | % Funcs  | % Lines  |
--------------|----------|----------|----------|----------|
All files     |   82.5   |   78.3   |   85.0   |   82.1   |
 user.service.ts    |   90.0   |   85.0   |   92.0   |   89.5   |
 user.repository.ts |   75.0   |   70.0   |   78.0   |   74.2   |
--------------|----------|----------|----------|----------|
```

### Watch Mode with Coverage

```bash
npm run test:watch -- --coverage
```

### Check Coverage for Specific File

```bash
npm test -- --testPathPattern=user.service --coverage
```

### Generate HTML Coverage Report

```bash
npm run test:cov
# Open coverage/index.html in browser
```

## Coverage Gaps

### Identifying Missing Coverage

```bash
npm run test:cov -- --verbose
```

Shows lines that aren't covered:

```
------ user.service.ts ------
14-18 | if (user.age > 65) return this.seniorDiscount();
21-25 | if (user.premium) calculatePremiumDiscount();
```

### Coverage Watermark (Baseline)

Set current coverage as baseline:

```javascript
// jest.config.js
const baseline = require('./coverage/coverage-summary.json').total;

module.exports = {
  coverageThreshold: {
    global: {
      branches: Math.ceil(baseline.branches.pct),
      functions: Math.ceil(baseline.functions.pct),
      lines: Math.ceil(baseline.lines.pct),
      statements: Math.ceil(baseline.statements.pct),
    },
  },
};
```

## Improving Coverage

### Finding Low Coverage Areas

```bash
npm run test:cov -- --coverageReporters=text-summary
```

### Common Low-Coverage Patterns

**Error handlers not tested:**
```typescript
// Low coverage - catch block not tested
try {
  await repository.save(data);
} catch (error) {
  logger.error('Failed to save', error);
  throw new DatabaseError('Save failed');
}
```

**Fix - Test error scenario:**
```typescript
it('should log error when save fails', async () => {
  mockRepository.save.mockRejectedValue(new Error('DB error'));

  await expect(service.create(data)).rejects.toThrow(DatabaseError);
  expect(logger.error).toHaveBeenCalled();
});
```

**Conditional code:**
```typescript
// Low coverage - else branch not tested
if (isPremium(user)) {
  return applyPremiumLogic(user);
} else {
  return applyBasicLogic(user);
}
```

**Fix - Test both paths:**
```typescript
it('should apply premium logic for premium users', async () => {
  const result = await service.process(premiumUser);
  expect(result).toHaveProperty('premium', true);
});

it('should apply basic logic for basic users', async () => {
  const result = await service.process(basicUser);
  expect(result).toHaveProperty('premium', false);
});
```

## Coverage Goals

### Recommended Minimums

| Layer | Minimum | Target |
|-------|---------|--------|
| Services | 80% | 90%+ |
| Repositories | 85% | 95%+ |
| Controllers | 70% | 80%+ |
| Utils/Helpers | 80% | 85%+ |
| Pipes/Guards | 75% | 85%+ |

### Files to Exclude

```javascript
collectCoverageFrom: [
  'src/**/*.ts',
  '!src/**/*.module.ts',      // Module wiring
  '!src/**/main.ts',          // Entry point
  '!src/**/*.entity.ts',      // Domain models (simple getters/setters)
  '!src/**/dto/**',           // DTOs (just data structures + decorators)
  '!src/**/*.interface.ts',   // Interfaces (type definitions only)
  '!src/config/**',           // Config loading
  '!src/common/decorators/**', // Decorators (framework setup)
]
```

## CI/CD Integration

### GitHub Actions Example

```yaml
# .github/workflows/test.yml
name: Tests

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
      - uses: actions/setup-node@v2
        with:
          node-version: '18'
      
      - run: npm install
      - run: npm test -- --coverage
      
      - name: Upload coverage
        uses: codecov/codecov-action@v2
        with:
          files: ./coverage/coverage-final.json
      
      - name: Check coverage
        run: |
          if [ $(cat coverage/coverage-summary.json | jq '.total.lines.pct') -lt 80 ]; then
            echo "Coverage below 80%"
            exit 1
          fi
```

## Coverage Best Practices

### DO's

- ✅ Test business logic thoroughly
- ✅ Test error conditions explicitly
- ✅ Test edge cases (empty, null, max values)
- ✅ Aim for high coverage on critical paths
- ✅ Use coverage as guidance, not a target
- ✅ Refactor to improve testability
- ✅ Set realistic thresholds per layer

### DON'Ts

- ❌ Don't test implementation details (mock internals)
- ❌ Don't write tests just to hit coverage targets
- ❌ Don't test framework code (NestJS decorators)
- ❌ Don't test trivial getters/setters
- ❌ Don't ignore low-coverage areas
- ❌ Don't set unrealistic thresholds (100% rarely needed)
- ❌ Don't sacrifice test quality for coverage

## Maintaining Coverage

### Pre-commit Hook

```bash
# .husky/pre-commit
npm test -- --coverage --coverageThreshold=./jest.config.js
```

### Prevent Coverage Regressions

```javascript
// jest.config.js - Don't let coverage decrease
const baseline = {
  branches: 75,
  functions: 80,
  lines: 80,
  statements: 80,
};

module.exports = {
  coverageThreshold: {
    global: baseline,
  },
};
```

### Track Coverage Over Time

Add to CI/CD:
```bash
echo "Current coverage:" && npm test -- --coverage --silent
```

Store results and compare across commits.
