---
name: feature-delivery
description: Orchestrate complete user story or feature delivery using NestJS best practices. Guides through planning, development, testing, code quality, security, documentation, performance optimization, and deployment using all project skills. Use when implementing a new feature, user story, or significant enhancement.
---

# Feature Delivery Workflow

This skill orchestrates the complete feature delivery process from planning through deployment, leveraging all project development skills to ensure production-grade quality, security, and maintainability.

## Complete Delivery Cycle

### Phase 1: Planning & Architecture

**Goal**: Define requirements, design the solution, and plan implementation.

**Checklist**:

- [ ] **Clarify Requirements**
  - Document user story acceptance criteria
  - Identify edge cases and error scenarios
  - List related features/dependencies

- [ ] **Design Architecture**
  - Use the **Architecture Skill**: Define domain, infrastructure, application, and presentation layers
  - Create module structure (see: *Code Conventions Skill*)
  - Plan database entities and relationships
  - Design API contracts (DTOs)
  
- [ ] **Security Planning**
  - Use the **Security Skill**: Identify authentication/authorization needs
  - Plan input validation requirements
  - Consider OWASP threats relevant to the feature

- [ ] **Performance Planning**
  - Use the **Performance Skill**: Identify caching opportunities
  - Plan for heavy operations (queues vs. real-time)
  - Design database indexes for common queries

- [ ] **Configuration Needs**
  - Use the **Configuration Skill**: List environment-specific settings
  - Plan secrets (API keys, credentials)
  - Document required `.env` variables

### Phase 2: Implementation

**Goal**: Build the feature following NestJS standards.

**Checklist**:

- [ ] **Create Module Structure**
  - Use the **Code Conventions Skill**: Create feature module, controller, service, repository
  - File naming: `feature.module.ts`, `feature.controller.ts`, `feature.service.ts`
  - Create DTOs in `dto/` folder: `create-feature.dto.ts`, `update-feature.dto.ts`

- [ ] **Implement Entities & Database**
  - Create entity file in `entities/` folder
  - Use the **Database Optimization Skill**: Add indexes for frequently queried fields
  - Create TypeORM migration if needed
  - Document schema design decisions

- [ ] **Implement Service Layer**
  - Use the **Architecture Skill**: Keep business logic in services only
  - Inject repositories, external services, and utilities
  - Handle errors using the **Error Handling Skill**: Custom error classes
  - Add structured logging using the **Error Handling Skill**: Logging patterns

- [ ] **Implement Controller Layer**
  - Use the **Code Conventions Skill**: Clear endpoint naming
  - Use the **Documentation Skill**: Add Swagger decorators
  - Add input validation using `class-validator`
  - Use the **Security Skill**: Add authentication/authorization guards

- [ ] **Implement Repository**
  - Use the **Database Optimization Skill**: Query optimization patterns
  - Prevent N+1 queries with eager loading
  - Use pagination for large result sets
  - Add custom query methods as needed

- [ ] **Handle Performance Requirements**
  - Use the **Performance Skill**: Add caching where appropriate
  - Implement queues for heavy operations
  - Optimize database queries (indexes, select, pagination)

- [ ] **Security Implementation**
  - Use the **Security Skill**: Implement guards, authentication, authorization
  - Validate all inputs
  - Use the **Configuration Skill**: Externalize secrets
  - Sanitize responses (don't expose internal details)

- [ ] **Documentation**
  - Use the **Documentation Skill**: Add Swagger/OpenAPI decorators
  - Document all endpoints, request/response models
  - Add examples to DTOs with `@ApiProperty` and descriptions

### Phase 3: Unit Testing

**Goal**: Test business logic with comprehensive coverage.

**Checklist**:

- [ ] **Service Tests**
  - Use the **Unit Testing Skill**: Create `feature.service.spec.ts`
  - Mock all dependencies (repositories, external services)
  - Test happy path and error scenarios
  - Ensure ≥70% coverage

- [ ] **Repository Tests**
  - Test query methods with test database
  - Use the **Unit Testing Skill**: Mocking Skill for complex scenarios
  - Test edge cases (empty results, large datasets)

- [ ] **Controller Tests**
  - Use the **Unit Testing Skill**: Test endpoint contracts
  - Mock services
  - Verify guards work (401, 403 responses)
  - Test input validation

- [ ] **Coverage Verification**
  - Use the **Unit Testing Skill**: Coverage Skill
  - Run: `npm run test:cov`
  - Target ≥70% coverage
  - Identify and test coverage gaps

### Phase 4: E2E Testing

**Goal**: Validate complete workflows with realistic data.

**Checklist**:

- [ ] **Environment Setup**
  - Use the **E2E Testing Skill**: Environment Skill
  - Configure test database (in-memory SQLite or test PostgreSQL)
  - Seed test data
  - Set up teardown/cleanup

- [ ] **API Contract Tests**
  - Use the **E2E Testing Skill**: Create `feature.e2e-spec.ts`
  - Test all endpoints with real requests
  - Verify request/response formats match Swagger docs
  - Test error responses (400, 403, 404, 500)

- [ ] **Workflow Tests**
  - Use the **E2E Testing Skill**: Advanced Skill
  - Test multi-step user workflows
  - Test concurrent operations
  - Test database transactions and rollback

- [ ] **Test Isolation**
  - Use the **E2E Testing Skill**: Database Skill
  - Clear database between tests
  - Use transactions for cleanup
  - Prevent test interdependencies

- [ ] **Run Full E2E Suite**
  - Run: `npm run test:e2e`
  - All tests pass before proceeding

### Phase 5: Code Quality & Security

**Goal**: Ensure code meets quality standards and is secure.

**Checklist**:

- [ ] **Linting & Formatting**
  - Use the **Repository Hygiene Skill**: Run linting
  - Run: `npm run lint:fix` to auto-fix issues
  - Run: `npm run format` to format code
  - All linting errors resolved

- [ ] **Type Safety**
  - Use the **Code Conventions Skill**: Run type checking
  - Run: `npm run type-check`
  - No TypeScript errors

- [ ] **Security Audit**
  - Use the **Security Skill**: Review against OWASP Top 10
  - No SQL injection vulnerabilities (use parameterized queries)
  - No sensitive data leaks in logs
  - Validate all external input

- [ ] **Error Handling**
  - Use the **Error Handling Skill**: Implement global filters
  - All errors logged consistently
  - No stack traces exposed in API responses
  - All edge cases handled

- [ ] **Documentation Quality**
  - Use the **Documentation Skill**: Verify Swagger accuracy
  - Run: `npm run swagger:generate` (if configured)
  - API docs match implementation
  - All request/response examples valid

### Phase 6: Repository Hygiene

**Goal**: Ensure code meets repository standards before merge.

**Checklist**:

- [ ] **Pre-Commit Checks**
  - Use the **Repository Hygiene Skill**: Hooks Skill
  - Hooks pass locally: linting, type-check, formatting
  - Run: `git commit` and verify hooks execute

- [ ] **Branch & PR Setup**
  - Create feature branch: `git checkout -b feature/descriptive-name`
  - Use the **Code Conventions Skill**: Commit messages
  - Follow conventional commit format: `feat(feature): description`

- [ ] **PR Requirements**
  - Create PR on GitHub
  - Link to issue/user story
  - Add description of changes

### Phase 7: CI/CD Pipeline

**Goal**: Verify all automated checks pass.

**Checklist**:

- [ ] **Workflow Execution**
  - Use the **Repository Hygiene Skill**: GitHub Actions Skill
  - All GitHub Actions workflows trigger automatically
  - Monitor workflow status in PR

- [ ] **Required Checks**
  - Type checking: ✓
  - Linting: ✓
  - Formatting: ✓
  - Unit tests (all Node versions): ✓
  - Unit test coverage (≥70%): ✓
  - E2E tests: ✓
  - Security audit: ✓
  - Build successful: ✓

- [ ] **Workflow Troubleshooting**
  - Use the **Repository Hygiene Skill**: Troubleshooting Skill
  - Review workflow logs if any fail
  - Fix issues and push again

### Phase 8: Performance & Monitoring

**Goal**: Ensure feature performs well under load.

**Checklist**:

- [ ] **Caching Validation**
  - Use the **Performance Skill**: Caching Skill
  - Verify cache hits for read-heavy operations
  - Check TTL values are appropriate

- [ ] **Database Optimization**
  - Use the **Performance Skill**: Database Skill
  - Verify indexes are in place
  - Check for N+1 query issues
  - Pagination implemented for large datasets

- [ ] **Queue Implementation**
  - Use the **Performance Skill**: Queuing Skill
  - Heavy operations use queues (emails, reports, etc.)
  - Retry logic configured
  - Dead letter queue set up

- [ ] **Monitoring & Alerts**
  - Use the **Performance Skill**: Monitoring Skill
  - Slow query logging enabled
  - Error rates monitored
  - Performance baseline established

### Phase 9: Deployment

**Goal**: Safely deploy feature to production.

**Checklist**:

- [ ] **Code Review & Approval**
  - Use the **Repository Hygiene Skill**: Branch Protection Skill
  - Minimum 1 approval required
  - All status checks passing
  - No merge conflicts

- [ ] **Pre-Deployment Verification**
  - Use the **Configuration Skill**: Environment Skill
  - Production `.env` configured correctly
  - Secrets configured in CI/CD/hosting platform
  - Database migrations ready

- [ ] **Deployment Execution**
  - Use the **Repository Hygiene Skill**: Deployment workflow
  - Merge to main (triggers deploy)
  - Monitor deployment logs
  - Verify feature is live

- [ ] **Post-Deployment Verification**
  - Use the **Error Handling Skill**: Monitoring Skill
  - Check application health endpoints
  - Verify feature endpoints working
  - Monitor error rates
  - Check performance metrics

### Phase 10: Documentation & Handoff

**Goal**: Document feature for team and future reference.

**Checklist**:

- [ ] **API Documentation**
  - Use the **Documentation Skill**: Sync Skill
  - OpenAPI spec generated
  - Swagger UI accessible
  - Documentation examples accurate

- [ ] **Architecture Documentation**
  - Use the **Architecture Skill**: Document module interactions
  - Explain design decisions
  - Document any deviations from standards

- [ ] **Configuration Documentation**
  - Use the **Configuration Skill**: Document all environment variables
  - Document required secrets
  - Add to `.env.example`

- [ ] **Operational Runbook**
  - Document common operations (cache invalidation, queue retry)
  - Document troubleshooting steps
  - Document monitoring/alerting

## Simple Example: User Registration Feature

**User Story**: "As a new user, I want to register an account with email/password so I can access the application."

### Phase 1: Planning
- **Requirements**: Email validation, password strength, duplicate prevention
- **Architecture**: Auth module with User entity, AuthService, AuthController
- **Security**: Password hashing (bcrypt), input validation
- **Performance**: Minimal, no heavy operations
- **Configuration**: JWT secret, password rules

### Phase 2-3: Implementation & Testing
```bash
# Create module structure
src/auth/
  ├── auth.module.ts
  ├── auth.controller.ts        # POST /auth/register
  ├── auth.service.ts           # registerUser method
  ├── auth.service.spec.ts
  ├── auth.repository.ts        # Query users by email
  └── dto/
      ├── register.dto.ts       # Swagger: email, password
      └── user-response.dto.ts  # Response without password
```

### Phase 4-5: Quality Checks
```bash
npm run lint                # ESLint passes
npm run format:check        # Prettier passes
npm run type-check          # TypeScript passes
npm run test:cov            # >70% coverage
npm run test:e2e            # Registration flow works
```

### Phase 6-9: Deploy
```bash
git commit -m "feat(auth): implement user registration"
# PR → review → CI checks pass → merge → deploy
```

## Complex Example: Order Processing Feature

**User Story**: "As a customer, I want to place an order so that my items are processed and shipped."

### Phase 1: Planning
- **Requirements**: Cart validation, payment processing, inventory update, order confirmation
- **Architecture**: Orders module, Payments service (external), Inventory service, NotificationService
- **Security**: User ownership, payment data handling
- **Performance**: Heavy operation (payment) → queue, caching for inventory
- **Configuration**: Stripe API key, mail service credentials

### Phase 2: Implementation
```bash
# Module structure
src/orders/
  ├── orders.module.ts
  ├── orders.controller.ts     # POST /orders, GET /orders/:id
  ├── orders.service.ts        # Complex business logic
  ├── orders.repository.ts
  ├── entities/
  │   ├── order.entity.ts      # With indexes on userId, status, createdAt
  │   └── order-item.entity.ts
  ├── dto/
  │   ├── create-order.dto.ts  # Cart items, shipping address
  │   └── order-response.dto.ts
  └── orders.service.spec.ts
```

### Phase 3: Service Implementation (with skills)
```typescript
// Orchestrates multiple skills:

// 1. Architecture: Keep business logic here
async createOrder(userId: string, createOrderDto: CreateOrderDto) {
  // Validate inventory (Architecture: domain logic)
  await this.validateInventory(createOrderDto.items);
  
  // Create order in DB (Database: repository pattern)
  const order = await this.ordersRepository.create(order);
  
  // Heavy operation → Queue (Performance: queuing skill)
  await this.paymentsQueue.add('process-payment', {
    orderId: order.id,
    amount: order.total,
  });
  
  // Cache result briefly (Performance: caching)
  await this.cache.set(`order:${order.id}`, order, 300);
  
  return order;
}

// 2. Error Handling: Custom errors
if (!inventory) {
  throw new OutOfStockError('Product not available');
}

// 3. Logging: Structured logs
this.logger.log(`Order created: ${order.id}`, { userId, total: order.total });
```

### Phase 4: E2E Testing
```bash
# Test complete workflows (E2E Testing: advanced skill)
- Register user
- Add to cart
- Create order (queues payment)
- Check order status
- Verify payment processed
- Confirm notification sent
```

### Phase 5: Quality & Security
```bash
# Security checks (Security skill):
- Input validation on order data
- User ownership verification
- No payment details logged
- CSRF protection

# Performance checks (Performance skill):
- Order queries indexed on userId, createdAt
- Inventory calls cached
- Payment processing async in queue
```

### Phase 6-9: Deploy to Production
```bash
# All 10 phases completed
# PR merged → GitHub Actions → Deploy → Live
```

## Skill Integration Map

| Phase | Primary Skills | Reference |
|-------|---|---|
| **1. Planning** | Architecture, Security, Performance, Configuration | architecture/SKILL.md |
| **2. Implementation** | Code Conventions, Architecture, Error Handling | code-conventions/SKILL.md |
| **3. Unit Testing** | Unit Testing, Mocking | unit-testing/SKILL.md |
| **4. E2E Testing** | E2E Testing, Database, Environment | e2e-testing/SKILL.md |
| **5. Code Quality** | Code Conventions, Repository Hygiene | repo-hygiene/SKILL.md |
| **6. Repository** | Repository Hygiene (hooks) | repo-hygiene/hooks.md |
| **7. CI/CD** | Repository Hygiene (GitHub Actions) | repo-hygiene/github-actions.md |
| **8. Performance** | Performance & Scalability | performance/SKILL.md |
| **9. Deployment** | Repository Hygiene, Configuration | repo-hygiene/SKILL.md |
| **10. Documentation** | Documentation, API Documentation | documentation/SKILL.md |

## Quick Reference Checklists

### Pre-Implementation (5 min)
- [ ] User story understood and refined
- [ ] Architecture sketched (layers, modules)
- [ ] Security threats identified
- [ ] Performance concerns noted

### Development (varies)
- [ ] Module structure created
- [ ] Entities designed with indexes
- [ ] Services implement business logic
- [ ] Controllers handle routing
- [ ] DTOs define contracts

### Testing (varies)
- [ ] Unit tests passing (≥70% coverage)
- [ ] E2E tests validate workflows
- [ ] No linting errors
- [ ] TypeScript strict mode passes

### Quality Gate (5 min)
- [ ] All automated checks pass
- [ ] Security audit complete
- [ ] Performance baseline met
- [ ] Documentation accurate

### Merge (5 min)
- [ ] Code review approved
- [ ] CI/CD pipeline green
- [ ] Ready for deployment
- [ ] Deployment plan documented

## Troubleshooting During Delivery

| Issue | Skill | Reference |
|-------|-------|-----------|
| Test coverage below threshold | Unit Testing | unit-testing/coverage.md |
| N+1 queries in E2E tests | Performance | performance/database.md |
| ESLint/Prettier conflicts | Repository Hygiene | repo-hygiene/SKILL.md |
| Security vulnerabilities | Security | security/attacks.md |
| Slow database queries | Performance | performance/database.md |
| CI/CD workflow failing | Repository Hygiene | repo-hygiene/troubleshooting.md |
| Cache invalidation issues | Performance | performance/caching.md |
| E2E test flakiness | E2E Testing | e2e-testing/database.md |

## Additional Resources

- For detailed architecture patterns, see [architecture.md](architecture.md)
- For skill selection guide, see [skill-matrix.md](skill-matrix.md)
- For delivery checklist templates, see [templates.md](templates.md)
