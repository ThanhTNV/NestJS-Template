---
name: repo-hygiene
description: Enforce repository quality standards using linting, formatting, testing, and CI/CD pipelines. Implement GitHub Actions workflows, pre-commit hooks, and branch protection rules to keep main deployable. Use when setting up CI/CD, configuring branch rules, or enforcing code quality checks.
---

# Repository Hygiene

This skill guides implementing and maintaining repository hygiene standards: automated code quality checks, testing requirements, and deployment safety mechanisms.

## Linting & Formatting

### ESLint Configuration

```bash
npm install --save-dev eslint @typescript-eslint/eslint-plugin @typescript-eslint/parser
npx eslint --init
```

```javascript
// eslint.config.mjs (Flat Config)
import typescriptPlugin from '@typescript-eslint/eslint-plugin';
import typescriptParser from '@typescript-eslint/parser';

export default [
  {
    ignores: ['node_modules', 'dist', 'coverage', '.nestjs-cache'],
  },
  {
    files: ['src/**/*.ts'],
    languageOptions: {
      parser: typescriptParser,
      parserOptions: {
        project: './tsconfig.json',
        sourceType: 'module',
      },
    },
    plugins: {
      '@typescript-eslint': typescriptPlugin,
    },
    rules: {
      '@typescript-eslint/explicit-function-return-types': 'error',
      '@typescript-eslint/no-unused-vars': ['error', { argsIgnorePattern: '^_' }],
      '@typescript-eslint/no-explicit-any': 'error',
      '@typescript-eslint/naming-convention': [
        'error',
        {
          selector: 'default',
          format: ['camelCase'],
        },
        {
          selector: 'typeLike',
          format: ['PascalCase'],
        },
      ],
      'no-console': ['warn', { allow: ['warn', 'error'] }],
      'prefer-const': 'error',
    },
  },
  {
    files: ['test/**/*.ts'],
    rules: {
      '@typescript-eslint/explicit-function-return-types': 'off',
    },
  },
];
```

### Prettier Configuration

```bash
npm install --save-dev prettier
```

```json
// .prettierrc
{
  "semi": true,
  "trailingComma": "es5",
  "singleQuote": true,
  "printWidth": 100,
  "tabWidth": 2,
  "useTabs": false,
  "arrowParens": "always",
  "endOfLine": "lf"
}
```

### npm Scripts

```json
{
  "scripts": {
    "lint": "eslint src/ test/",
    "lint:fix": "eslint src/ test/ --fix",
    "format": "prettier --write \"src/**/*.ts\" \"test/**/*.ts\"",
    "format:check": "prettier --check \"src/**/*.ts\" \"test/**/*.ts\"",
    "type-check": "tsc --noEmit",
    "type-check:watch": "tsc --noEmit --watch"
  }
}
```

## Pre-Commit Hooks

### Husky Setup

```bash
npm install husky --save-dev
npx husky install
npx husky add .husky/pre-commit "npm run format:check && npm run lint && npm run type-check"
```

```bash
#!/bin/sh
# .husky/pre-commit
npm run format:check
npm run lint
npm run type-check
```

### Lint-Staged for Staged Files

```bash
npm install lint-staged --save-dev
```

```json
// package.json
{
  "lint-staged": {
    "src/**/*.ts": ["prettier --write", "eslint --fix", "tsc --noEmit"],
    "test/**/*.ts": ["prettier --write", "eslint --fix"]
  }
}
```

```bash
#!/bin/sh
# .husky/pre-commit
npx lint-staged
```

## Testing Requirements

### Jest Configuration

```typescript
// jest.config.js
export default {
  moduleFileExtensions: ['js', 'json', 'ts'],
  rootDir: 'src',
  testRegex: '.*\\.spec\\.ts$',
  transform: {
    '^.+\\.(t|j)s$': 'ts-jest',
  },
  collectCoverageFrom: [
    '**/*.(t|j)s',
    '!**/*.module.ts',
    '!**/index.ts',
  ],
  coverageDirectory: '../coverage',
  testEnvironment: 'node',
  coverageThreshold: {
    global: {
      branches: 70,
      functions: 70,
      lines: 70,
      statements: 70,
    },
  },
};
```

### npm Scripts

```json
{
  "scripts": {
    "test": "jest",
    "test:watch": "jest --watch",
    "test:cov": "jest --coverage",
    "test:debug": "node --inspect-brk -r tsconfig-paths/register -r ts-node/register node_modules/.bin/jest --runInBand",
    "test:e2e": "jest --config ./test/jest-e2e.json"
  }
}
```

## GitHub Actions CI/CD

### Main Test Workflow

```yaml
# .github/workflows/test.yml
name: Test

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main, develop]

permissions:
  contents: read
  statuses: write

jobs:
  test:
    name: Test & Coverage
    runs-on: ubuntu-latest

    strategy:
      matrix:
        node-version: [18.x, 20.x]

    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node-version }}
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Type check
        run: npm run type-check

      - name: Lint
        run: npm run lint

      - name: Format check
        run: npm run format:check

      - name: Run unit tests
        run: npm run test:cov

      - name: Upload coverage
        uses: codecov/codecov-action@v3
        with:
          files: ./coverage/coverage-final.json
          flags: unittests

      - name: Check coverage threshold
        run: |
          COVERAGE=$(cat coverage/coverage-summary.json | grep -o '"lines":{"total":[0-9]*' | grep -o '[0-9]*$')
          if [ $COVERAGE -lt 70 ]; then
            echo "❌ Coverage below 70%: $COVERAGE%"
            exit 1
          fi
          echo "✅ Coverage: $COVERAGE%"
```

### E2E Test Workflow

```yaml
# .github/workflows/e2e.yml
name: E2E Tests

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main, develop]

jobs:
  e2e:
    name: End-to-End Tests
    runs-on: ubuntu-latest

    services:
      postgres:
        image: postgres:15
        env:
          POSTGRES_USER: test
          POSTGRES_PASSWORD: test
          POSTGRES_DB: test_db
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 5432:5432

    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: 20.x
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Run migrations
        env:
          DATABASE_URL: postgresql://test:test@localhost:5432/test_db
        run: npm run typeorm migration:run

      - name: Run E2E tests
        env:
          DATABASE_URL: postgresql://test:test@localhost:5432/test_db
          NODE_ENV: test
        run: npm run test:e2e

      - name: Upload E2E results
        if: always()
        uses: actions/upload-artifact@v3
        with:
          name: e2e-results
          path: test/results/
```

### Security & Code Quality

```yaml
# .github/workflows/quality.yml
name: Code Quality

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main, develop]

jobs:
  security:
    name: Security Audit
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20.x
          cache: 'npm'
      - name: Audit dependencies
        run: npm audit --audit-level=moderate

  dependencies:
    name: Dependency Check
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Dependency review
        uses: actions/dependency-review-action@v3
        if: github.event_name == 'pull_request'

  build:
    name: Build Check
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20.x
          cache: 'npm'
      - name: Install dependencies
        run: npm ci
      - name: Build
        run: npm run build
      - name: Upload build artifacts
        uses: actions/upload-artifact@v3
        with:
          name: build
          path: dist/
```

## Branch Protection Rules

Configure in GitHub:

**Settings → Branches → Add rule**

```yaml
# Branch name pattern: main

# Protect matching branches:
✓ Require pull request reviews before merging
  - Required approving reviews: 1
  - Require review from code owners: ✓
  - Dismiss stale pull request approvals: ✓

✓ Require status checks to pass before merging
  - Test (ubuntu-latest, 18.x)
  - Test (ubuntu-latest, 20.x)
  - E2E Tests
  - Code Quality
  - Security Audit

✓ Require branches to be up to date before merging

✓ Require code review before merging

✓ Require conversation resolution before merging

✓ Require signed commits

✓ Dismiss stale pull request approvals when new commits are pushed

✓ Restrict who can push to matching branches
  - Include administrators: false

✓ Allow force pushes: None

✓ Allow deletions: false
```

## Preventing Broken Deployments

### Deployment Prevention

```yaml
# .github/workflows/deploy.yml
name: Deploy

on:
  push:
    branches: [main]

jobs:
  test:
    name: Pre-Deployment Tests
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20.x
          cache: 'npm'
      - run: npm ci
      - run: npm run lint
      - run: npm run type-check
      - run: npm run test:cov
      - run: npm run test:e2e

  build:
    name: Build for Deployment
    needs: test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20.x
          cache: 'npm'
      - run: npm ci
      - run: npm run build

  deploy:
    name: Deploy to Production
    needs: [test, build]
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    steps:
      - uses: actions/checkout@v4
      - name: Deploy
        run: |
          echo "Deploying to production..."
          # Add your deployment logic here
      - name: Notify deployment
        run: echo "✅ Deployment successful"
```

### PR Check Requirements

**All must pass before merge:**

1. ✓ Linting (ESLint)
2. ✓ Formatting (Prettier)
3. ✓ Type checking (TypeScript)
4. ✓ Unit tests (Jest)
5. ✓ E2E tests (SuperTest)
6. ✓ Coverage threshold (70%)
7. ✓ Build successful

## Maintenance & Monitoring

### GitHub Status Page

```bash
# Check latest workflow runs
gh run list --branch main --status completed --limit 10

# View specific workflow
gh run view <run-id>

# Trigger manual workflow
gh workflow run test.yml -r main
```

### Protected Branch Status

```bash
# List branch protection rules
gh repo rule list

# Check branch status
gh api repos/{owner}/{repo}/branches/main --jq '.protection'
```

## Implementation Checklist

### Initial Setup

- [ ] Create `.eslintrc` / `eslint.config.mjs`
- [ ] Create `.prettierrc`
- [ ] Add npm scripts (lint, format, test)
- [ ] Install husky: `npx husky install`
- [ ] Add pre-commit hook
- [ ] Create GitHub Actions workflows
- [ ] Configure branch protection rules
- [ ] Enable required status checks
- [ ] Test PR workflow (create test PR)

### Ongoing Maintenance

- [ ] Review and update linting rules quarterly
- [ ] Update dependencies regularly (`npm audit fix`)
- [ ] Monitor test coverage trends
- [ ] Review failed workflows
- [ ] Keep CI/CD yaml synced with tooling changes
- [ ] Document any exceptions or overrides

## Review Checklist

When reviewing repository configuration:

- [ ] All required checks pass before merge
- [ ] Main branch has branch protection enabled
- [ ] Coverage threshold enforced (≥70%)
- [ ] Pre-commit hooks prevent common issues
- [ ] GitHub Actions workflows run on all PRs
- [ ] Deployment workflow requires all tests to pass
- [ ] Security audit included in CI
- [ ] Force push disabled on main
- [ ] Admin approvals required for exceptions
- [ ] Build artifacts uploaded/retained
- [ ] E2E tests run against real database

## Additional Resources

- For GitHub Actions workflows and advanced setups, see [github-actions.md](github-actions.md)
- For pre-commit hook configuration and local enforcement, see [hooks.md](hooks.md)
- For troubleshooting and common issues, see [troubleshooting.md](troubleshooting.md)
