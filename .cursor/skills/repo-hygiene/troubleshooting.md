# Troubleshooting & Common Issues

This document covers common repository hygiene issues and solutions.

## Linting Issues

### ESLint Not Finding Files

```bash
# Problem: "No files matching the pattern"

# Solution 1: Check file paths
npx eslint src/ test/

# Solution 2: Verify glob pattern
npx eslint "src/**/*.ts"

# Solution 3: Check .eslintignore
cat .eslintignore

# Solution 4: List files explicitly
npx eslint src/main.ts src/app.module.ts
```

### TypeScript Errors in ESLint

```bash
# Problem: "Cannot find type definition"

# Solution: Ensure tsconfig is correct
npm run type-check

# Verify parserOptions
# eslint.config.mjs should have:
# parserOptions: { project: './tsconfig.json' }

# Rebuild node_modules
rm -rf node_modules package-lock.json
npm install
```

### Conflicting Rules

```bash
# Problem: "Conflicting rules" or lint-fix creates infinite loops

# Solution: Check for conflicting Prettier + ESLint rules
npm install eslint-config-prettier --save-dev

# Update eslint.config.mjs:
import prettierConfig from 'eslint-config-prettier';

export default [
  // ... other configs
  prettierConfig,  // Must be last
];
```

## Formatting Issues

### Prettier Changes Not Applied

```bash
# Problem: "prettier --fix" changes nothing

# Solution 1: Check configuration
cat .prettierrc

# Solution 2: Verify file types
prettier --list-different "src/**/*.ts"

# Solution 3: Check for ignored files
cat .prettierignore

# Solution 4: Force re-format
prettier --write --ignore-unknown "src/**/*.ts"
```

### Line Ending Issues

```bash
# Problem: Mixed CRLF/LF causing format changes

# Solution: Set global git config
git config --global core.autocrlf input  # Mac/Linux
git config --global core.autocrlf true   # Windows

# Then normalize repo
git add --renormalize -A
git commit -m "chore: normalize line endings"
```

## Test Failures

### Jest Coverage Threshold Fails

```bash
# Problem: "Coverage threshold not met"

# Solution 1: Check current coverage
npm run test:cov

# View coverage report
open coverage/lcov-report/index.html

# Solution 2: Identify uncovered lines
# Look for lines without coverage in report

# Solution 3: Lower threshold temporarily (if needed)
// jest.config.js
coverageThreshold: {
  global: {
    branches: 60,    // Was 70
    functions: 60,
    lines: 60,
    statements: 60,
  },
},

# Solution 4: Write missing tests
npm run test -- --coverage --watch
```

### E2E Tests Failing

```bash
# Problem: "Database connection refused"

# Solution 1: Check if services are running
docker ps | grep postgres
# or
npm run db:start

# Solution 2: Verify connection string
echo $DATABASE_URL

# Solution 3: Check migrations ran
npm run typeorm migration:show

# Solution 4: Reset test database
npm run db:reset:test

# Solution 5: Run with debug output
DEBUG=* npm run test:e2e
```

### Timeout Errors

```bash
# Problem: "Test timeout after 5000ms"

# Solution 1: Increase timeout
jest.setTimeout(10000);  // In test file

// Or in jest.config.js
testTimeout: 10000,

# Solution 2: Check what's slow
console.time('database-query');
// ... code ...
console.timeEnd('database-query');

# Solution 3: Mock external services
jest.mock('@nestjs/axios', () => ({
  HttpService: jest.fn(),
}));
```

## CI/CD Pipeline Issues

### Workflow Not Triggering

```bash
# Problem: "Workflow not running"

# Solution 1: Check trigger conditions
cat .github/workflows/test.yml | grep -A 5 "^on:"

# Solution 2: Verify branch matches
# - push to main/develop only?
# - PR from specific branches?

# Solution 3: Check if disabled
# Settings → Actions → Disable/Enable workflows

# Solution 4: Check commit author
# Some workflows skip bot commits

# Solution 5: Manual trigger if supported
gh workflow run test.yml -r main
```

### Status Check Not Showing

```bash
# Problem: "Workflow doesn't appear in branch protection"

# Solution 1: Ensure job completes successfully
# Check workflow logs

# Solution 2: Verify job names match exactly
# GitHub Settings → Branch rules → Status checks
# Must match: name + context

# Solution 3: Workflow must have run on main first
# Workflows don't show until they complete

# Solution 4: Check permissions
# Workflow needs write permission to status
permissions:
  statuses: write
```

### Out of Memory in CI

```bash
# Problem: "JavaScript heap out of memory"

# Solution 1: Increase Node memory
env:
  NODE_OPTIONS: --max-old-space-size=4096

# Solution 2: Run tests separately (not parallel)
strategy:
  matrix:
    node-version: [18.x, 20.x]
  max-parallel: 1  # Run sequentially

# Solution 3: Skip coverage in CI (if not needed)
npm test -- --coverage=false
```

## Pre-Commit Hook Issues

### Hooks Not Running

```bash
# Problem: "Git hooks don't execute"

# Solution 1: Check husky installation
npm list husky
ls -la .husky/pre-commit

# Solution 2: Make hooks executable
chmod +x .husky/pre-commit
chmod +x .husky/pre-push

# Solution 3: Reinstall husky
npm uninstall husky
npm install husky --save-dev
npx husky install

# Solution 4: Check Node in PATH
which node
# Should output path, not empty
```

### Hook Fails Silently

```bash
# Problem: "Hook runs but doesn't block commits"

# Solution 1: Verify exit code
#!/bin/sh
npm run test || exit 1
#                  ^^^^^^ Important!

# Solution 2: Add debug output
#!/bin/sh
set -e  # Exit on any error
set -x  # Print commands before running

npm run lint
npm run type-check

# Solution 3: Check for trap commands
# Some shells need explicit error handling
trap 'exit 1' ERR
```

### Slow Pre-Commit Hooks

```bash
# Problem: "Hooks take too long, blocking commits"

# Solution 1: Only check staged files
npx lint-staged  # Only changed files

# Solution 2: Skip full test suite
# Use pre-commit for linting, pre-push for tests

# Solution 3: Add progress output
echo "⏳ Running tests (this may take a minute)..."

# Solution 4: Allow bypass for urgent commits
git commit --no-verify  # Emergency only!
```

## Common Git Issues

### Cannot Push to Main

```bash
# Problem: "Your branch is ahead of origin by X commits"

# Solution 1: Check branch protection
# Need PR approval? Can't force push?
git log origin/main..HEAD

# Solution 2: Create PR instead
git push origin my-branch
# Then create PR on GitHub

# Solution 3: If allowed, force push (careful!)
git push --force-with-lease origin main
```

### Branch Protection Requires Status Checks

```bash
# Problem: "Branch protection rule violation"

# Solution 1: Wait for workflow to complete
gh run watch <run-id>

# Solution 2: Check workflow logs
gh run view <run-id> --log

# Solution 3: Retry workflow
gh run rerun <run-id> --failed

# Solution 4: Temporarily disable protection (if admin)
# Settings → Branches → Temporarily disable
```

## Dependency Issues

### npm audit Fails

```bash
# Problem: "npm audit: X vulnerabilities found"

# Solution 1: Update vulnerable packages
npm audit fix

# Solution 2: Interactive fix
npm audit fix --force

# Solution 3: Check what's vulnerable
npm audit

# Solution 4: Accept risk (if necessary)
npm install --audit-level=low  # Allow low severity

# Solution 5: Add to CI ignore
# .npmrc
audit-level=moderate  # Only fail on moderate+
```

### Node Modules Size Growing

```bash
# Problem: "npm install gets slower"

# Solution 1: Clean cache
npm cache clean --force

# Solution 2: Use ci instead of install
npm ci  # Cleaner install

# Solution 3: Prune dev dependencies in production
npm install --production

# Solution 4: Check for duplicates
npm dedupe
```

## Debugging Workflow Failures

### Enable Debug Logging

```yaml
# In GitHub Actions workflow
- name: Debug
  env:
    ACTIONS_STEP_DEBUG: true
  run: npm test
```

### Local Workflow Testing

```bash
# Test workflow locally with act
brew install act  # macOS
# or
choco install act  # Windows

# Run workflow
act -j test

# Run with environment
act -j test -s DATABASE_URL=postgresql://...
```

### Check GitHub Actions Quota

```bash
# View usage
gh api user/actions/repositories \
  --jq '.total_repositories_used'

# View costs
gh api repos/OWNER/REPO/actions/usage
```

## Performance Optimization

### Speed Up Linting

```bash
# 1. Cache eslint results
npx eslint --cache src/

# 2. Lint only changed files
npx eslint $(git diff --name-only --cached)

# 3. Use parallel processing
npm run lint -- --max-warnings 0

# 4. Skip full type check in pre-commit
# Type check in CI instead
```

### Faster Tests

```bash
# 1. Run tests in band (faster on CI)
npm test -- --runInBand

# 2. Skip coverage in pre-commit
npm test -- --coverage=false

# 3. Mock slow services
jest.mock('@nestjs/axios');

# 4. Use test.only for development
test.only('specific test', () => {
  // Only this test runs
});
```

### Faster Workflows

```yaml
# 1. Use concurrency (cancel old runs)
concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

# 2. Cache dependencies
- uses: actions/setup-node@v4
  with:
    cache: 'npm'

# 3. Run jobs in parallel
jobs:
  lint:
    # ...
  test:
    # ...
  build:
    needs: [lint, test]
    # ...
```

## Getting Help

### Useful Commands

```bash
# Check what would be committed
git diff --cached

# View recent commits
git log --oneline -10

# Check branch status
git status

# View workflow runs
gh run list

# View specific run
gh run view <run-id> --log

# Trigger manual workflow
gh workflow run test.yml -r main

# List configured hooks
git config --local --list | grep hook
```

### Create Minimal Reproduction

```bash
# If issue is unclear, create minimal repo
git init test-repo
cd test-repo
npm init -y
npm install --save-dev eslint prettier jest

# Add minimal eslint config
echo "export default [];" > eslint.config.mjs

# Test the issue
npx eslint src/
```
