# Skill Selection Matrix

This document helps you quickly identify which skills to use during feature delivery.

## By Phase

### Phase 1: Planning & Design

| Question | Skill | When to Use |
|----------|-------|------------|
| How should layers be organized? | **Architecture** | Designing module structure |
| What entities and relationships? | **Code Conventions** | Database schema planning |
| What are the security risks? | **Security** | User data, authentication, authorization |
| Will this be slow? Should we cache? | **Performance** | Heavy computations, frequent access |
| Need external services/APIs? | **Configuration** | Managing API keys, credentials |
| How to handle errors? | **Error Handling** | Exception strategy |

### Phase 2-3: Implementation & Unit Testing

| Task | Skill | Reference |
|------|-------|-----------|
| Create module structure | **Code Conventions** | code-conventions/structure.md |
| Name files/classes | **Code Conventions** | code-conventions/SKILL.md |
| Write service business logic | **Architecture** | architecture/patterns.md |
| Implement repository queries | **Performance** | performance/database.md |
| Mock dependencies in tests | **Unit Testing** | unit-testing/mocking.md |
| Achieve coverage targets | **Unit Testing** | unit-testing/coverage.md |
| Log errors properly | **Error Handling** | error-handling/logging.md |
| Throw custom exceptions | **Error Handling** | error-handling/custom-errors.md |
| Add validation | **Security** | security/SKILL.md |
| Implement authentication | **Security** | security/authentication.md |
| Implement authorization | **Security** | security/authorization.md |
| Cache query results | **Performance** | performance/caching.md |
| Queue heavy operations | **Performance** | performance/queuing.md |
| Document endpoints | **Documentation** | documentation/SKILL.md |
| Add Swagger examples | **Documentation** | documentation/patterns.md |

### Phase 4: E2E Testing

| Scenario | Skill | When to Use |
|----------|-------|------------|
| Set up test database | **E2E Testing** | e2e-testing/environment.md |
| Prevent database pollution | **E2E Testing** | e2e-testing/database.md |
| Test complete workflows | **E2E Testing** | e2e-testing/SKILL.md |
| Test auth flows | **E2E Testing** | e2e-testing/advanced.md |
| Test concurrent operations | **E2E Testing** | e2e-testing/advanced.md |

### Phase 5: Code Quality

| Check | Skill | When to Use |
|-------|-------|------------|
| ESLint, Prettier | **Repository Hygiene** | repo-hygiene/SKILL.md |
| TypeScript strict mode | **Repository Hygiene** | repo-hygiene/SKILL.md |
| Coverage thresholds | **Unit Testing** | unit-testing/coverage.md |
| OWASP compliance | **Security** | security/attacks.md |
| Performance baseline | **Performance** | performance/SKILL.md |

### Phase 6-7: Repository & CI/CD

| Task | Skill | When to Use |
|------|-------|------------|
| Set up pre-commit hooks | **Repository Hygiene** | repo-hygiene/hooks.md |
| Configure GitHub Actions | **Repository Hygiene** | repo-hygiene/github-actions.md |
| Set branch protection | **Repository Hygiene** | repo-hygiene/SKILL.md |
| Troubleshoot failures | **Repository Hygiene** | repo-hygiene/troubleshooting.md |

### Phase 8-10: Performance, Deployment & Docs

| Task | Skill | When to Use |
|------|-------|------------|
| Monitor performance | **Performance** | performance/SKILL.md |
| Set up monitoring | **Error Handling** | error-handling/monitoring.md |
| Configure environment | **Configuration** | configuration/SKILL.md |
| Manage secrets | **Configuration** | configuration/secrets.md |
| Generate API docs | **Documentation** | documentation/SKILL.md |
| Sync specs | **Documentation** | documentation/sync.md |

## By Feature Type

### Simple CRUD (User, Product, Category)

**Skills Used**: Code Conventions, Unit Testing, Documentation

**Focus**:
- Simple entity with basic CRUD
- Minimal business logic
- Standard testing

**Checklist**:
- [ ] Code Conventions: Module structure
- [ ] Code Conventions: Naming
- [ ] Unit Testing: Service tests
- [ ] Documentation: Swagger decorators
- [ ] Repository Hygiene: Linting passes

---

### Authentication (Login, Registration)

**Skills Used**: Security, Error Handling, Configuration

**Focus**:
- Password hashing (bcrypt)
- JWT tokens
- Guards and strategies
- Error handling for auth failures

**Checklist**:
- [ ] Security: JWT setup
- [ ] Security: Password hashing
- [ ] Configuration: JWT secret in .env
- [ ] Error Handling: Auth error classes
- [ ] Unit Testing: Auth service tests
- [ ] E2E Testing: Login/register flows

---

### External Service Integration (Payment, Email)

**Skills Used**: Performance (queues), Error Handling, Security

**Focus**:
- Queue for async processing
- Error handling for service failures
- Sensitive data protection
- Retries and dead letter queue

**Checklist**:
- [ ] Performance: Queue configuration
- [ ] Error Handling: Custom errors for service failures
- [ ] Configuration: API credentials in .env
- [ ] Security: No credentials in logs
- [ ] Unit Testing: Mock external service
- [ ] E2E Testing: Test failure scenarios

---

### High-Traffic Data Access (Dashboard, Reporting)

**Skills Used**: Performance (caching, database), E2E Testing

**Focus**:
- Database indexes
- Query optimization
- Caching strategies
- N+1 prevention

**Checklist**:
- [ ] Performance: Database indexes
- [ ] Performance: Query optimization
- [ ] Performance: Caching strategy
- [ ] E2E Testing: Performance baseline
- [ ] Unit Testing: Mock heavy queries

---

### Complex Workflow (Order Processing, Approval Flows)

**Skills Used**: Architecture, E2E Testing, Performance (events)

**Focus**:
- Multi-step orchestration
- State management
- Error recovery
- Event-driven coordination

**Checklist**:
- [ ] Architecture: Layer separation
- [ ] Performance: Queue for state changes
- [ ] Error Handling: State rollback on failure
- [ ] E2E Testing: Multi-step workflows
- [ ] Unit Testing: Each step isolated

---

### Real-Time Features (WebSockets, Notifications)

**Skills Used**: Performance, Error Handling, Configuration

**Focus**:
- Queue for notifications
- Connection management
- Error recovery
- Monitoring

**Checklist**:
- [ ] Performance: Queue for messages
- [ ] Configuration: WebSocket settings
- [ ] Error Handling: Connection failures
- [ ] E2E Testing: Real-time scenarios

---

## By Common Issues

### Issue: Tests Run Slowly

**Use**: Performance Skill (Database Skill)

**Solution**: 
- Database indexes for test queries
- Use in-memory database for unit tests
- Batch test data seeding

---

### Issue: Coverage Below Threshold

**Use**: Unit Testing Skill (Coverage Skill)

**Solution**:
- Identify uncovered branches
- Mock complex dependencies
- Test error scenarios

---

### Issue: N+1 Queries Detected

**Use**: Performance Skill (Database Skill)

**Solution**:
- Eager loading with `relations`
- Batch loading
- QueryBuilder `select`

---

### Issue: Cache Inconsistency

**Use**: Performance Skill (Caching Skill)

**Solution**:
- Invalidate on data changes
- Use event-based invalidation
- Set appropriate TTLs

---

### Issue: E2E Test Flakiness

**Use**: E2E Testing Skill (Database Skill)

**Solution**:
- Isolation between tests
- Transaction cleanup
- Avoid timing dependencies

---

### Issue: Security Vulnerability Found

**Use**: Security Skill (Attacks Skill)

**Solution**:
- Validate all inputs
- Parameterized queries
- Sanitize responses

---

### Issue: CI/CD Pipeline Failing

**Use**: Repository Hygiene Skill (Troubleshooting Skill)

**Solution**:
- Check workflow logs
- Verify status check names
- Ensure all local checks pass

---

### Issue: Environment Variables Not Working

**Use**: Configuration Skill (Environments Skill)

**Solution**:
- Verify `.env.example` matches `.env`
- Check environment priority
- Verify CI/CD secrets set

---

## Quick Navigation

**I'm implementing...**

- [ ] **A new REST endpoint** → Code Conventions + Documentation
- [ ] **User authentication** → Security + Error Handling
- [ ] **Database queries** → Performance + Unit Testing
- [ ] **External API call** → Performance (queues) + Error Handling
- [ ] **Email/notification** → Performance (queues) + Configuration
- [ ] **Caching strategy** → Performance (caching)
- [ ] **Complex workflow** → Architecture + E2E Testing
- [ ] **Sensitive data** → Security + Configuration
- [ ] **High-traffic feature** → Performance + Error Handling + Monitoring

**I need to troubleshoot...**

- [ ] **Slow tests** → Performance or E2E Testing (environment)
- [ ] **Low coverage** → Unit Testing (coverage)
- [ ] **Database issues** → Performance (database)
- [ ] **Failed CI** → Repository Hygiene (troubleshooting)
- [ ] **Config issues** → Configuration (environments)
- [ ] **Security concern** → Security (attacks)
- [ ] **Performance problem** → Performance (monitoring)

## Skill Dependencies

```
Feature Delivery orchestrates all skills:

┌─────────────────────────────────────────────────────────────┐
│                    Feature Delivery                         │
│              (Planning → Development → Deploy)              │
└─────────────────────────────────────────────────────────────┘
            ↓
    ┌───────────────────────────────────────────────────┐
    │  Architecture          Code Conventions           │
    │  Security              Error Handling             │
    │  Performance           Configuration              │
    │  Documentation                                    │
    └───────────────────────────────────────────────────┘
            ↓
    ┌───────────────────────────────────────────────────┐
    │  Unit Testing          E2E Testing                │
    │  Repository Hygiene (CI/CD)                       │
    └───────────────────────────────────────────────────┘
```

## When to Use Multiple Skills

### Example: User Registration

```
Phase 1 (Planning):
  → Architecture: Module structure
  → Security: Password hashing strategy
  → Configuration: JWT secret

Phase 2 (Implementation):
  → Code Conventions: File structure
  → Security: Input validation
  → Error Handling: Auth errors
  → Documentation: Swagger decorators

Phase 3 (Testing):
  → Unit Testing: Service, controller tests
  → E2E Testing: Complete registration flow

Phase 4 (Quality):
  → Repository Hygiene: Linting, formatting
  → All checks pass

Phase 5 (Deploy):
  → Configuration: Production secrets
  → Repository Hygiene: Branch protection
  → All CI/CD checks green
```

### Example: Order Processing

```
Phase 1 (Planning):
  → Architecture: Multi-layer design
  → Performance: Caching + queues
  → Security: User authorization
  → Error Handling: Failure scenarios

Phase 2 (Implementation):
  → Code Conventions: Module structure
  → Architecture: Service orchestration
  → Performance: Database + caching
  → Error Handling: Error classes + logging
  → Security: Input validation
  → Documentation: API documentation

Phase 3 (Testing):
  → Unit Testing: Each component
  → E2E Testing: Complete order flow + concurrent orders

Phase 4 (Quality):
  → Performance: Query optimization + monitoring
  → Security: OWASP review
  → Repository Hygiene: All checks

Phase 5 (Deploy):
  → Configuration: Environment setup
  → All systems operational
```
