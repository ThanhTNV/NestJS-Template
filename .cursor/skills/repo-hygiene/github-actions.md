# GitHub Actions Workflows

This document covers advanced GitHub Actions patterns for NestJS projects.

## Workflow Structure

### Basic Workflow Anatomy

```yaml
name: Workflow Name                    # Display name in GitHub UI

on:                                    # Trigger conditions
  push:
    branches: [main, develop]
  pull_request:
    branches: [main, develop]

permissions:                           # GitHub token permissions
  contents: read
  statuses: write
  pull-requests: write

jobs:
  job-name:                           # Job identifier
    name: Display Name
    runs-on: ubuntu-latest            # Runner type
    
    strategy:                         # Matrix strategy
      matrix:
        node-version: [18.x, 20.x]
    
    steps:                            # Job steps
      - uses: actions/checkout@v4     # Pre-built action
      - name: Step name
        run: npm run test              # Shell command
```

## Advanced Patterns

### 1. Conditional Steps

```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20.x
      - run: npm ci

      # Only on push to main
      - name: Build for production
        if: github.event_name == 'push' && github.ref == 'refs/heads/main'
        run: npm run build:prod

      # Only on PR
      - name: Comment on PR
        if: github.event_name == 'pull_request'
        uses: actions/github-script@v7
        with:
          script: |
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: '✅ All checks passed!'
            })

      # Only if previous step failed
      - name: Report failure
        if: failure()
        run: echo "Tests failed"

      # Always run (even if previous steps fail)
      - name: Cleanup
        if: always()
        run: npm run cleanup
```

### 2. Outputs and Artifacts

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    outputs:
      version: ${{ steps.version.outputs.value }}
      
    steps:
      - uses: actions/checkout@v4
      
      - name: Get version
        id: version
        run: echo "value=$(cat package.json | grep version | head -1 | awk '{print $2}')" >> $GITHUB_OUTPUT
      
      - name: Build
        run: npm run build
      
      - name: Upload artifact
        uses: actions/upload-artifact@v3
        with:
          name: build-${{ steps.version.outputs.value }}
          path: dist/
          retention-days: 7

  deploy:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - name: Download artifact
        uses: actions/download-artifact@v3
        with:
          name: build-${{ needs.build.outputs.version }}
      
      - name: Deploy
        run: |
          echo "Deploying version ${{ needs.build.outputs.version }}"
```

### 3. Matrix Strategies

```yaml
jobs:
  test:
    strategy:
      matrix:
        node-version: [18.x, 20.x]
        os: [ubuntu-latest, macos-latest, windows-latest]
        include:
          # Add specific combinations
          - node-version: 20.x
            os: ubuntu-latest
            coverage: true
        exclude:
          # Skip certain combinations
          - node-version: 18.x
            os: windows-latest
    
    runs-on: ${{ matrix.os }}
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node-version }}
      - run: npm ci
      - run: npm test
      
      - name: Upload coverage
        if: matrix.coverage
        uses: codecov/codecov-action@v3
```

### 4. Container Services

```yaml
jobs:
  integration-tests:
    runs-on: ubuntu-latest
    
    services:
      postgres:
        image: postgres:15-alpine
        env:
          POSTGRES_USER: testuser
          POSTGRES_PASSWORD: testpass
          POSTGRES_DB: testdb
        options: >-
          --health-cmd "pg_isready -U testuser"
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 5432:5432
      
      redis:
        image: redis:7-alpine
        options: >-
          --health-cmd "redis-cli ping"
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 6379:6379
    
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20.x
      - run: npm ci
      
      - name: Run migrations
        env:
          DATABASE_URL: postgresql://testuser:testpass@localhost:5432/testdb
        run: npm run typeorm migration:run
      
      - name: Run tests
        env:
          DATABASE_URL: postgresql://testuser:testpass@localhost:5432/testdb
          REDIS_URL: redis://localhost:6379
        run: npm test
```

### 5. Scheduled Workflows

```yaml
# .github/workflows/scheduled-checks.yml
name: Scheduled Checks

on:
  schedule:
    # Every day at 2 AM UTC
    - cron: '0 2 * * *'
  # Also allow manual trigger
  workflow_dispatch:

jobs:
  dependency-check:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20.x
      
      - name: Update dependencies
        run: npm update
      
      - name: Run tests
        run: npm test
      
      - name: Create PR if updates available
        if: steps.dependency-check.outputs.has-updates == 'true'
        uses: peter-evans/create-pull-request@v5
        with:
          commit-message: 'chore: update dependencies'
          title: 'chore: update dependencies'
          body: 'Automated dependency update'
          branch: automated-dependency-updates
```

### 6. Secrets Management

```yaml
jobs:
  deploy:
    runs-on: ubuntu-latest
    environment:
      name: production
      url: https://api.example.com
    steps:
      - uses: actions/checkout@v4
      
      # Access repository secrets
      - name: Deploy
        env:
          AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
          AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          DATABASE_URL: ${{ secrets.DATABASE_URL }}
        run: npm run deploy
```

### 7. Reusable Workflows

```yaml
# .github/workflows/test.yml (Main workflow)
name: Test

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  test:
    uses: ./.github/workflows/test-reusable.yml
    secrets: inherit

# .github/workflows/test-reusable.yml (Reusable workflow)
name: Reusable Test Workflow

on:
  workflow_call:

jobs:
  run-tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20.x
      - run: npm ci
      - run: npm test
```

## Complete Workflow Example

```yaml
# .github/workflows/complete-ci.yml
name: Complete CI Pipeline

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main, develop]

permissions:
  contents: read
  statuses: write
  pull-requests: write
  checks: write

jobs:
  # Stage 1: Quality Checks
  quality:
    name: Code Quality
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20.x
          cache: 'npm'
      
      - run: npm ci
      - run: npm run type-check
      - run: npm run lint
      - run: npm run format:check

  # Stage 2: Unit Tests
  unit-tests:
    name: Unit Tests
    needs: quality
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
      
      - run: npm ci
      - run: npm run test:cov
      
      - name: Upload coverage
        uses: codecov/codecov-action@v3
        if: matrix.node-version == '20.x'

  # Stage 3: Build
  build:
    name: Build
    needs: [quality, unit-tests]
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20.x
          cache: 'npm'
      
      - run: npm ci
      - run: npm run build
      
      - uses: actions/upload-artifact@v3
        with:
          name: build
          path: dist/
          retention-days: 5

  # Stage 4: E2E Tests
  e2e:
    name: E2E Tests
    needs: build
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:15-alpine
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
      
      - run: npm ci
      
      - name: Setup test database
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

  # Stage 5: Security
  security:
    name: Security Audit
    needs: quality
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20.x
          cache: 'npm'
      
      - run: npm ci
      - run: npm audit --audit-level=moderate
      
      - name: Dependency review
        uses: actions/dependency-review-action@v3
        if: github.event_name == 'pull_request'

  # Final: Status
  status:
    name: CI Status
    needs: [quality, unit-tests, build, e2e, security]
    runs-on: ubuntu-latest
    if: always()
    steps:
      - name: Determine status
        run: |
          if [ "${{ needs.quality.result }}" == "failure" ] || \
             [ "${{ needs.unit-tests.result }}" == "failure" ] || \
             [ "${{ needs.build.result }}" == "failure" ] || \
             [ "${{ needs.e2e.result }}" == "failure" ] || \
             [ "${{ needs.security.result }}" == "failure" ]; then
            echo "❌ Pipeline failed"
            exit 1
          fi
          echo "✅ All checks passed"
```

## Optimization Tips

### 1. Use Caching

```yaml
- uses: actions/setup-node@v4
  with:
    node-version: 20.x
    cache: 'npm'  # Auto-cache node_modules
```

### 2. Cancel Previous Runs

```yaml
concurrency:
  group: ${{ github.workflow }}-${{ github.head_ref || github.ref }}
  cancel-in-progress: true
```

### 3. Fail Fast

```yaml
strategy:
  fail-fast: true
  matrix:
    node-version: [18.x, 20.x]
```

### 4. Reduce Runner Time

```yaml
- name: Quick lint (changed files only)
  run: npm run lint -- --changed-files-only
```

## Troubleshooting

### Workflow Failures

```bash
# View workflow run details
gh run view <run-id> --log

# Re-run failed jobs
gh run rerun <run-id>

# Watch live
gh run watch <run-id>
```

### Debugging

```yaml
- name: Debug info
  if: failure()
  run: |
    echo "Node version:"
    node --version
    echo "npm version:"
    npm --version
    echo "Environment:"
    env | grep -E '^(NODE|NPM|PATH)'
```
