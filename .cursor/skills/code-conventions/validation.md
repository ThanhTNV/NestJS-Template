# Validation Scripts

This document describes the convention validation approach for the project.

## Running Validation

### All Checks

```bash
npm run validate:conventions
```

This runs:
1. ESLint for code quality
2. Prettier for formatting
3. Structure validation
4. Naming convention checks

### Individual Checks

```bash
# Lint check
npm run lint

# Format check
npm run format:check

# Naming convention check
npm run validate:naming

# Folder structure check
npm run validate:structure
```

## What Gets Validated

### ESLint Checks

- No unused variables
- Consistent naming conventions
- Proper error handling
- Security best practices

### Prettier Checks

- Consistent indentation (2 spaces)
- Quote style (double quotes)
- Semicolon usage
- Line length limits

### Structure Validation

- Controllers exist in `src/<feature>/`
- Services colocated with controllers
- Repositories follow naming pattern
- DTOs in `src/<feature>/dto/`
- Entities in `src/<feature>/entities/`
- Unit tests use `.spec.ts` suffix
- No mixed concerns in files

### Naming Validation

- Controllers: `<name>.controller.ts`
- Services: `<name>.service.ts`
- Repositories: `<name>.repository.ts`
- Modules: `<name>.module.ts`
- DTOs: `create-<name>.dto.ts`, `update-<name>.dto.ts`
- Entities: `<name>.entity.ts`
- Classes match file names in PascalCase

## Pre-commit Hook

The project uses Husky to enforce validation before commits:

```bash
# Setup (one-time)
npm install husky --save-dev
npx husky install

# Add pre-commit hook
npx husky add .husky/pre-commit "npm run validate:conventions"
```

## Fixing Violations

### Auto-fixable Issues

```bash
npm run lint:fix
npm run format
```

### Manual Fixes

Review the violation output and manually adjust:

1. **Rename files** to match conventions
2. **Move code** to correct layers
3. **Add DTOs** for validation
4. **Update imports** after file moves

## Integrating into CI/CD

Add to your pipeline (e.g., `.github/workflows/ci.yml`):

```yaml
- name: Validate conventions
  run: npm run validate:conventions
```

This ensures all PRs meet consistency standards before merge.
