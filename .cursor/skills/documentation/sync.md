# Keeping Specs in Sync with Code

This document covers strategies for maintaining OpenAPI spec alignment with source code.

## Problem: Spec Drift

**Without synchronization:**
```
Week 1: Code and Swagger in sync ✓
Week 2: Developer updates endpoint, forgets to update Swagger ❌
Week 3: Spec shows old fields that no longer exist
Week 4: Developers trust docs less, rely on reading code instead 📉
```

## Solution 1: Manual Sync Checklist

### Pre-commit Validation

Add to `.husky/pre-commit`:

```bash
#!/bin/bash
# Validate Swagger documentation

echo "Checking API documentation..."

# Check that all controllers have @ApiTags
if ! grep -r "@ApiTags" src/*/\*.controller.ts > /dev/null; then
  echo "❌ Missing @ApiTags in some controllers"
  exit 1
fi

# Check that all endpoints have @ApiOperation
if grep -r "@Get\|@Post\|@Put\|@Delete" src/*/\*.controller.ts | \
   grep -v "@ApiOperation" > /dev/null; then
  echo "❌ Some endpoints missing @ApiOperation"
  exit 1
fi

# Check that all DTOs have @ApiProperty
if grep -r "class.*Dto" src/*/dto/ | \
   grep -v "@ApiProperty" > /dev/null; then
  echo "❌ Some DTO fields missing @ApiProperty"
  exit 1
fi

echo "✓ Documentation checks passed"
```

### CI/CD Validation

```yaml
# .github/workflows/validate-docs.yml
name: Validate Documentation

on: [pull_request]

jobs:
  docs:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
      - uses: actions/setup-node@v2

      - run: npm install

      - name: Check Swagger decorators
        run: npm run validate:docs

      - name: Compare generated OpenAPI spec
        run: npm run validate:spec-diff
```

## Solution 2: Automated Sync Tools

### Generate HTML Documentation

```bash
# npm script in package.json
{
  "scripts": {
    "generate-docs": "swagger-cli bundle src/openapi.yaml --outfile dist/openapi-generated.json",
    "validate:docs": "npm run generate-docs"
  }
}
```

### TypeDoc Swagger Plugin

```typescript
// Extract JSDoc comments automatically
// (Requires typescript-to-swagger plugin setup)

/**
 * Get user by ID
 * @param id - User unique identifier
 * @returns User object
 * @throws NotFoundException if user doesn't exist
 */
@Get(':id')
async getUser(@Param('id') id: string): Promise<UserResponseDto> {
  return this.userService.findById(id);
}
```

### Generate OpenAPI from Code

```bash
#!/bin/bash
# scripts/validate-spec.sh

# Generate new spec
npm run generate-docs

# Compare with tracked spec
if ! diff -q dist/openapi.json src/openapi.json > /dev/null; then
  echo "⚠️  OpenAPI spec is out of sync with code"
  echo "Run: npm run generate-docs && git add src/openapi.json"
  exit 1
fi

echo "✓ OpenAPI spec is in sync"
```

## Solution 3: Contract Testing

### OpenAPI Schema Validation

```typescript
// tests/openapi.spec.ts
import { Test } from '@nestjs/testing';
import { SwaggerModule, DocumentBuilder } from '@nestjs/swagger';
import * as fs from 'fs';

describe('OpenAPI Schema', () => {
  let app;

  beforeAll(async () => {
    const module = await Test.createTestingModule({
      imports: [AppModule],
    }).compile();

    app = module.createNestApplication();
    await app.init();
  });

  it('should have all endpoints documented', () => {
    const config = new DocumentBuilder().build();
    const document = SwaggerModule.createDocument(app, config);

    // Check all routes are documented
    const routes = document.paths;
    expect(Object.keys(routes).length).toBeGreaterThan(0);
  });

  it('should match tracked OpenAPI spec', () => {
    const config = new DocumentBuilder().build();
    const document = SwaggerModule.createDocument(app, config);

    const tracked = JSON.parse(
      fs.readFileSync('src/openapi.json', 'utf-8')
    );

    expect(document).toEqual(tracked);
  });

  it('should validate all responses have examples', () => {
    const config = new DocumentBuilder().build();
    const document = SwaggerModule.createDocument(app, config);

    Object.entries(document.paths).forEach(([path, methods]: [string, any]) => {
      Object.entries(methods).forEach(([method, operation]: [string, any]) => {
        if (operation.responses) {
          Object.values(operation.responses).forEach((response: any) => {
            expect(response.content?.['application/json']?.example).toBeDefined(
              `Missing example for ${method.toUpperCase()} ${path}`
            );
          });
        }
      });
    });
  });
});
```

## Solution 4: Versioning Strategy

### Track OpenAPI Changes

```bash
# .github/workflows/track-api-changes.yml
- name: Comment API changes
  if: github.event_name == 'pull_request'
  run: |
    npm run generate-docs
    git diff src/openapi.json > api-changes.txt
    echo "## API Changes" >> $GITHUB_STEP_SUMMARY
    cat api-changes.txt >> $GITHUB_STEP_SUMMARY
```

### Semantic Versioning

```typescript
// Update version when API changes
const config = new DocumentBuilder()
  .setTitle('MyApp API')
  .setVersion('1.2.0')  // Bump version on API changes
  .build();
```

## Solution 5: Documentation as Test

### Response Shape Validation

```typescript
// tests/response-shapes.spec.ts
describe('Response Shapes Match Documentation', () => {
  it('GET /users should return array of documented shape', async () => {
    const response = await request(app.getHttpServer())
      .get('/users')
      .expect(200);

    // Validate against Swagger schema
    const schema = openapi.paths['/users'].get.responses['200'].schema;
    expect(response.body).toMatchSchema(schema);
  });

  it('POST /users response should match UserResponseDto', async () => {
    const response = await request(app.getHttpServer())
      .post('/users')
      .send(createUserDto)
      .expect(201);

    // Should have documented fields
    expect(response.body).toHaveProperty('id');
    expect(response.body).toHaveProperty('email');
    expect(response.body).toHaveProperty('name');
    expect(response.body).toHaveProperty('createdAt');
  });
});
```

## Recommended Approach

### Combine Multiple Strategies

1. **Pre-commit**: Local validation (fast feedback)
2. **CI/CD**: Block merge if docs don't match (enforcement)
3. **Contract Tests**: Verify responses match docs (confidence)
4. **Tracking**: Track changes over time (accountability)

### Implementation Order

```bash
# Phase 1: Validation
npm run validate:docs      # Check decorators present

# Phase 2: Generation
npm run generate-docs      # Create spec from code

# Phase 3: Testing
npm run test:contracts     # Verify responses match spec

# Phase 4: CI/CD Gate
git push  # CI blocks if any step fails
```

## Maintenance Checklist

- [ ] Update Swagger when adding endpoint
- [ ] Update Swagger when changing request/response shape
- [ ] Run `npm run validate:docs` before commit
- [ ] Run `npm run test:contracts` to verify
- [ ] Review `api-changes.txt` in PRs
- [ ] Bump API version on breaking changes
- [ ] Track deprecated endpoints 30 days before removal

## Common Mistakes

### ❌ Missing `@ApiProperty` on DTO Fields

```typescript
// BAD - Not documented
export class CreateUserDto {
  email: string;
  name: string;
  age: number;
}

// GOOD - All fields documented
export class CreateUserDto {
  @ApiProperty({ description: 'User email', example: 'john@example.com' })
  email: string;

  @ApiProperty({ description: 'User name', example: 'John Doe' })
  name: string;

  @ApiProperty({ description: 'User age', example: 30 })
  age: number;
}
```

### ❌ No Error Response Documentation

```typescript
// BAD - Only success documented
@ApiResponse({ status: 200, type: UserResponseDto })
@Post()
async createUser(@Body() dto: CreateUserDto) { }

// GOOD - Success and errors documented
@ApiResponse({ status: 201, type: UserResponseDto })
@ApiResponse({ status: 400, description: 'Invalid input' })
@ApiResponse({ status: 409, description: 'Email already exists' })
@Post()
async createUser(@Body() dto: CreateUserDto) { }
```

### ❌ Fake or Outdated Examples

```typescript
// BAD - Example doesn't match actual behavior
@ApiProperty({
  example: 'user123', // But real IDs are user_abc123xyz
})
id: string;

// GOOD - Realistic examples
@ApiProperty({
  example: 'user_abc123xyz',
})
id: string;
```

## Tools & Resources

- `@nestjs/swagger` - Swagger integration
- `swagger-cli` - OpenAPI bundling
- `swagger-ui-express` - UI rendering
- `openapi-typescript` - Generate TypeScript types from spec
- `prism` - API mocking from OpenAPI spec
