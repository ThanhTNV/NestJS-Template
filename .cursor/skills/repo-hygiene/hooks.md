# Pre-Commit Hooks & Local Enforcement

This document covers Git hooks, husky setup, and local quality checks.

## Husky Setup

### Installation

```bash
# Install husky
npm install husky --save-dev

# Initialize husky
npx husky install

# Make sure husky can run (add to package.json)
# "postinstall": "husky install"
```

### Pre-Commit Hook

Create `.husky/pre-commit`:

```bash
#!/bin/sh
. "$(dirname "$0")/_/husky.sh"

echo "🔍 Running pre-commit checks..."

# Type checking
echo "📝 Type checking..."
npm run type-check || (echo "❌ Type check failed"; exit 1)

# Linting
echo "✨ Linting..."
npm run lint || (echo "❌ Linting failed"; exit 1)

# Formatting
echo "🎨 Checking format..."
npm run format:check || (echo "❌ Formatting issues found"; exit 1)

echo "✅ All pre-commit checks passed!"
```

### Pre-Push Hook

Create `.husky/pre-push`:

```bash
#!/bin/sh
. "$(dirname "$0")/_/husky.sh"

echo "🚀 Running pre-push checks..."

# Run all tests before push
echo "🧪 Running tests..."
npm run test -- --silent || (echo "❌ Tests failed"; exit 1)

# Check coverage
echo "📊 Checking coverage..."
npm run test:cov -- --silent || (echo "❌ Coverage check failed"; exit 1)

echo "✅ Ready to push!"
```

### Commit Message Hook

Create `.husky/commit-msg`:

```bash
#!/bin/sh
. "$(dirname "$0")/_/husky.sh"

# Validate commit message format
msg=$(cat "$1")

# Pattern: <type>(<scope>): <subject>
# Example: feat(auth): add login endpoint
if ! echo "$msg" | grep -E "^(feat|fix|docs|style|refactor|perf|test|chore)(\(.+\))?: .+" > /dev/null; then
  echo "❌ Invalid commit message format"
  echo "Expected: <type>(<scope>): <subject>"
  echo "Example: feat(auth): add login endpoint"
  exit 1
fi

# Check message length
if [ ${#msg} -gt 100 ]; then
  echo "⚠️  Commit message is too long (${#msg} > 100 chars)"
  echo "Consider making it more concise"
fi
```

## Lint-Staged

### Configuration

```bash
npm install lint-staged --save-dev
```

```json
// package.json
{
  "lint-staged": {
    "src/**/*.ts": [
      "prettier --write",
      "eslint --fix",
      "tsc --noEmit --skipLibCheck"
    ],
    "test/**/*.ts": [
      "prettier --write",
      "eslint --fix"
    ],
    "*.json": [
      "prettier --write"
    ]
  },
  "scripts": {
    "pre-commit": "lint-staged"
  }
}
```

Update `.husky/pre-commit`:

```bash
#!/bin/sh
. "$(dirname "$0")/_/husky.sh"

echo "🔍 Running lint-staged on changed files..."
npx lint-staged || (echo "❌ Lint-staged failed"; exit 1)

echo "✅ Pre-commit checks passed!"
```

## .git/hooks Fallback

If husky isn't available, create direct hooks:

```bash
# Create hooks directory
mkdir -p .git/hooks

# Make hooks executable
chmod +x .git/hooks/pre-commit
chmod +x .git/hooks/pre-push
chmod +x .git/hooks/commit-msg
```

## Pre-Commit Configuration (Alternative)

Python-based pre-commit framework:

```bash
pip install pre-commit
```

```yaml
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/pre-commit/mirrors-node-dev
    rev: v18.0.0
    hooks:
      - id: prettier
        args: [--write]
      - id: eslint
        args: [--fix]

  - repo: local
    hooks:
      - id: type-check
        name: Type check
        entry: npm run type-check
        language: system
        pass_filenames: false
```

Initialize:

```bash
pre-commit install
pre-commit install --hook-type commit-msg
```

## Skipping Hooks

### Emergency Skip

```bash
# Skip all hooks for single commit
git commit --no-verify -m "emergency fix"

# Skip pre-commit only
git commit --no-verify

# Skip pre-push only
git push --no-verify
```

### Conditional Skipping

```bash
# Skip if environment variable set
if [ -z "$SKIP_HOOKS" ]; then
  npm run type-check || exit 1
fi
```

## Hook Patterns

### 1. Format & Fix

```bash
#!/bin/sh
. "$(dirname "$0")/_/husky.sh"

# Auto-fix formatting
echo "🎨 Auto-fixing formatting..."
npm run format

# Auto-fix linting issues
echo "✨ Auto-fixing lint issues..."
npm run lint:fix

# Stage fixed files
git add -A

echo "✅ Files auto-fixed and staged"
```

### 2. Test Subset

```bash
#!/bin/sh
. "$(dirname "$0")/_/husky.sh"

# Get changed files
CHANGED_FILES=$(git diff --cached --name-only --diff-filter=ACM)

# Run tests only for changed files
echo "🧪 Testing changed files..."
npm run test -- $CHANGED_FILES || exit 1
```

### 3. Prevent Secrets

```bash
#!/bin/sh
. "$(dirname "$0")/_/husky.sh"

echo "🔒 Checking for secrets..."

# Check for common secret patterns
if git diff --cached -S "PRIVATE_KEY\|SECRET\|PASSWORD" > /dev/null; then
  echo "❌ Potential secrets found in staged changes"
  exit 1
fi

# Check for accidentally committed .env
if git diff --cached --name-only | grep -E '\.env(\.local)?$' > /dev/null; then
  echo "❌ .env file is being committed"
  exit 1
fi

echo "✅ No secrets detected"
```

### 4. Branch Naming

```bash
#!/bin/sh
# .git/hooks/prepare-commit-msg

BRANCH=$(git symbolic-ref --short HEAD)

# Validate branch name: feature/*, bugfix/*, hotfix/*, docs/*
if ! [[ $BRANCH =~ ^(feature|bugfix|hotfix|docs|chore)\/[a-z0-9\-]+$ ]]; then
  echo "❌ Invalid branch name: $BRANCH"
  echo "Expected format: <type>/<description>"
  echo "Example: feature/user-authentication"
  exit 1
fi
```

## Troubleshooting

### Hooks Not Running

```bash
# Check husky is installed
npm list husky

# Check hooks are executable
ls -la .husky/

# Verify husky init
cat .husky/.gitignore

# Reinstall
npx husky uninstall
npx husky install
```

### Hook Permissions (Windows)

```bash
# Make hooks executable
chmod +x .husky/*
chmod +x .husky/_/*
```

### Bypass for CI

Hooks should only run locally; CI uses GitHub Actions:

```bash
# In CI environment, skip hooks
git config core.hooksPath /dev/null

# Or use CI environment variable
if [ "$CI" != "true" ]; then
  npm run type-check || exit 1
fi
```

## Best Practices

### 1. Fast Feedback

Keep hooks fast (<5 seconds):

```bash
# ✅ GOOD - Fast checks
npm run lint:fix
npm run format:check

# ❌ SLOW - Full test suite
npm run test  # Avoid in pre-commit
```

### 2. Auto-Fix When Possible

```bash
# Don't just fail, fix it
npm run format:fix
npm run lint:fix
git add -A
```

### 3. Clear Error Messages

```bash
echo "❌ Type checking failed:"
echo "Run: npm run type-check"
echo "Or: npm run type-check --watch"
exit 1
```

### 4. Respect .gitignore

```bash
# Only check staged files
npx lint-staged

# Don't check entire repo
npm run lint  # Checks all files
```

## Recommended Hook Flows

### Development Setup

```
pre-commit hook:
  ↓
lint-staged (changed files only)
  ↓
Type check
  ↓
Format check
  ↓
✅ Pass: commit allowed
❌ Fail: commit blocked
```

### Pre-Push

```
pre-push hook:
  ↓
Full test suite
  ↓
Coverage check
  ↓
✅ Pass: push allowed
❌ Fail: push blocked
```

### Commit Message

```
commit-msg hook:
  ↓
Validate format (feat(scope): message)
  ↓
Check length
  ↓
Verify issue references
  ↓
✅ Pass: commit created
❌ Fail: commit aborted
```
