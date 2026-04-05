# Feature Delivery Checklist Templates

Ready-to-use checklists for different feature types.

## Universal Checklist

Use this for any feature:

```markdown
# Feature: [Feature Name]
# User Story: [US-XXX] [Description]
# Developer: [Name]
# Date Started: [Date]

## Phase 1: Planning & Design
- [ ] Requirements clarified and documented
- [ ] Architecture designed (layers, modules)
- [ ] Security threats identified
- [ ] Performance concerns addressed
- [ ] Database schema designed
- [ ] Configuration needs identified
- [ ] Acceptance criteria understood

## Phase 2: Implementation
- [ ] Module structure created
  - [ ] `[feature].module.ts`
  - [ ] `[feature].controller.ts`
  - [ ] `[feature].service.ts`
  - [ ] `[feature].repository.ts`
  - [ ] `dto/` folder with DTOs
  - [ ] `entities/` folder with entities
- [ ] Controllers implement endpoints
- [ ] Services implement business logic
- [ ] Repositories implement queries
- [ ] Error handling implemented
- [ ] Logging added at key points
- [ ] Input validation implemented
- [ ] Authentication guards added
- [ ] Authorization rules enforced
- [ ] Documentation (Swagger) added

## Phase 3: Unit Testing
- [ ] Service tests passing: ✓
- [ ] Controller tests passing: ✓
- [ ] Repository tests passing: ✓
- [ ] Coverage ≥70%: ✓
  - Branches: [X]%
  - Functions: [X]%
  - Lines: [X]%
  - Statements: [X]%

## Phase 4: E2E Testing
- [ ] Test database set up
- [ ] API contract tests passing
- [ ] Complete workflow tests passing
- [ ] Error scenario tests passing
- [ ] Concurrent operation tests passing

## Phase 5: Code Quality & Security
- [ ] ESLint: ✓ (0 errors)
- [ ] Prettier: ✓ (formatted)
- [ ] TypeScript: ✓ (strict mode)
- [ ] Security audit: ✓
  - [ ] Input validation
  - [ ] SQL injection prevention
  - [ ] XSS prevention
  - [ ] CSRF protection (if needed)
  - [ ] No sensitive data in logs
- [ ] Error handling: ✓
- [ ] Documentation: ✓

## Phase 6: Repository Setup
- [ ] Feature branch created: `feature/[descriptive-name]`
- [ ] Pre-commit hooks pass locally
- [ ] Code formatted and linted
- [ ] All tests passing locally

## Phase 7: CI/CD Verification
- [ ] GitHub Actions workflows triggered
- [ ] Type checking: ✓
- [ ] Linting: ✓
- [ ] Formatting: ✓
- [ ] Unit tests (Node 18.x): ✓
- [ ] Unit tests (Node 20.x): ✓
- [ ] Unit test coverage: ✓
- [ ] E2E tests: ✓
- [ ] Security audit: ✓
- [ ] Build successful: ✓

## Phase 8: Performance & Monitoring
- [ ] Database indexes in place
- [ ] N+1 queries prevented
- [ ] Caching strategy implemented
- [ ] Heavy operations queued
- [ ] Performance baseline met
- [ ] Monitoring/logging configured

## Phase 9: Deployment
- [ ] Code review approved
- [ ] All CI checks green
- [ ] Configuration prepared (production `.env`)
- [ ] Secrets configured in CI/CD
- [ ] Deployment executed
- [ ] Feature live and working

## Phase 10: Documentation & Handoff
- [ ] API documentation complete
- [ ] Swagger/OpenAPI spec accurate
- [ ] Architecture documented
- [ ] Configuration documented
- [ ] Operational runbook created
- [ ] Team notified

## Sign-Off
- Developer: _________________ Date: _______
- Reviewer: _________________ Date: _______
```

## CRUD Feature Template

```markdown
# CRUD Feature: [Entity Name]

## Requirements
- [ ] List endpoint (GET /[entities])
- [ ] Get single endpoint (GET /[entities]/:id)
- [ ] Create endpoint (POST /[entities])
- [ ] Update endpoint (PATCH /[entities]/:id)
- [ ] Delete endpoint (DELETE /[entities]/:id)
- [ ] Filtering/Pagination on list

## Implementation
- [ ] Entity created with proper types
- [ ] Indexes on commonly filtered fields
- [ ] Repository with CRUD methods
- [ ] Service with CRUD methods
- [ ] Controller with CRUD endpoints
- [ ] DTOs: CreateDto, UpdateDto, ResponseDto
- [ ] Input validation on create/update
- [ ] Authorization (owner check on update/delete)

## Testing
- [ ] Create: successful creation, validation errors
- [ ] Read: by ID, list with pagination
- [ ] Update: successful update, validation, ownership check
- [ ] Delete: successful deletion, ownership check
- [ ] Edge cases: not found, permission denied

## Quality
- [ ] ESLint/Prettier/TypeScript pass
- [ ] Coverage ≥70%
- [ ] E2E workflow tests pass
- [ ] Swagger docs accurate

## Deployment
- [ ] Database migration applied
- [ ] All checks pass
- [ ] Deployed to production
```

## Authentication Feature Template

```markdown
# Authentication Feature: [Method]

## Requirements
- [ ] User registration
- [ ] User login
- [ ] Token refresh (if JWT)
- [ ] Logout/token revocation
- [ ] Password reset (if applicable)

## Security
- [ ] Password hashing (bcrypt)
- [ ] JWT secret in environment
- [ ] Token expiration set
- [ ] Refresh token strategy defined
- [ ] CSRF protection (if session-based)
- [ ] Rate limiting on auth endpoints

## Implementation
- [ ] AuthService with auth methods
- [ ] AuthController with endpoints
- [ ] AuthGuard for protected routes
- [ ] Custom error classes for auth failures
- [ ] JWT strategy configured
- [ ] DTOs for auth payloads

## Testing
- [ ] Registration: success, validation, duplicate prevention
- [ ] Login: success, invalid credentials, user not found
- [ ] Token refresh: valid token, expired token, invalid token
- [ ] Protected routes: unauthorized, authorized
- [ ] E2E complete auth flow

## Documentation
- [ ] Swagger auth methods documented
- [ ] Example curl commands
- [ ] Token format explained

## Deployment
- [ ] All tests passing
- [ ] Secrets configured in CI/CD
- [ ] Auth working in production
```

## External Service Integration Template

```markdown
# Integration: [Service Name]

## Service Details
- Service: [e.g., Stripe, SendGrid]
- Endpoint: [API base URL]
- Authentication: [API key, OAuth, etc.]

## Implementation
- [ ] Service wrapper created
- [ ] API calls async in queue (if heavy)
- [ ] Error handling for service failures
- [ ] Retry logic with exponential backoff
- [ ] Dead letter queue for failed operations
- [ ] Credentials in environment variables

## Testing
- [ ] Unit tests with mocked service
- [ ] E2E tests with test account
- [ ] Failure scenarios tested
- [ ] Retry logic verified

## Monitoring
- [ ] Service health checks
- [ ] Failure rate monitoring
- [ ] Dead letter queue monitored
- [ ] Alerts configured

## Deployment
- [ ] API credentials configured
- [ ] Service account active
- [ ] Queue infrastructure running
- [ ] Tests passing against live service
```

## Complex Workflow Template

```markdown
# Workflow: [Workflow Name]

## Steps
1. [ ] Step 1: [Description]
2. [ ] Step 2: [Description]
3. [ ] Step 3: [Description]
4. [ ] Step N: [Description]

## State Management
- [ ] States defined: [list states]
- [ ] State transitions documented
- [ ] Invalid transitions prevented
- [ ] State changes logged

## Error Handling
- [ ] Step 1 failure: [recovery strategy]
- [ ] Step 2 failure: [recovery strategy]
- [ ] Step N failure: [recovery strategy]
- [ ] Complete rollback strategy

## Implementation
- [ ] Workflow service created
- [ ] Each step in separate method
- [ ] Transaction handling
- [ ] Compensating transactions for rollback
- [ ] Event emitters for state changes
- [ ] Logging at each step

## Testing
- [ ] Happy path E2E test
- [ ] Each step failure tested
- [ ] Recovery verified
- [ ] Concurrent workflows tested
- [ ] State consistency verified

## Deployment
- [ ] All tests passing
- [ ] Monitoring configured
- [ ] Alerting for failures
```

## High-Performance Feature Template

```markdown
# High-Performance Feature: [Name]

## Performance Requirements
- Response time target: [ms]
- Throughput target: [requests/sec]
- Data volume: [records/queries]

## Optimization Strategy
- [ ] Database indexes: [list]
- [ ] Caching strategy: [read-through/write-through/event-based]
- [ ] Query optimization: [eager loading/pagination/selection]
- [ ] Async operations: [list what goes to queue]
- [ ] Connection pooling: [pool size]

## Implementation
- [ ] Indexes created in schema
- [ ] Queries optimized with QueryBuilder
- [ ] Caching decorator/service implemented
- [ ] Heavy ops moved to queues
- [ ] Connection pool configured

## Testing
- [ ] Load test: [tool/approach]
- [ ] Baseline performance met
- [ ] Cache hit rate: [target]%
- [ ] Query execution time: <[ms]
- [ ] Queue processing time acceptable

## Monitoring
- [ ] Query performance dashboard
- [ ] Cache hit rate monitored
- [ ] Response time SLA monitored
- [ ] Database connection pool monitored
- [ ] Alert thresholds set

## Deployment
- [ ] All checks passing
- [ ] Performance baseline confirmed
- [ ] Monitoring active
- [ ] Ready for production traffic
```

## Migration/Data-Heavy Feature Template

```markdown
# Data Feature: [Operation]

## Requirements
- [ ] Data migration scope defined
- [ ] Rollback strategy
- [ ] Data validation rules
- [ ] Timeline/schedule

## Implementation
- [ ] TypeORM migration created
- [ ] Data transformation logic tested
- [ ] Validation queries written
- [ ] Rollback procedure documented
- [ ] Performance tested with real data size

## Pre-Deployment
- [ ] Backup verified
- [ ] Migration script tested locally
- [ ] Test run on staging
- [ ] Rollback tested
- [ ] Communication sent to team

## Deployment
- [ ] Backup taken
- [ ] Migration run
- [ ] Data validation passed
- [ ] Rollback procedure ready (if needed)
- [ ] Application still working

## Post-Deployment
- [ ] Data integrity verified
- [ ] Performance baseline checked
- [ ] User testing passed
- [ ] Documentation updated
```

## Customization Notes

- Replace `[placeholders]` with actual values
- Add/remove items as needed for your feature
- Link to related documentation
- Include dates and sign-offs
- Keep updated as progress is made
- Reference in PR description and commit messages
