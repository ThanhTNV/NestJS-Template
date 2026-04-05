<p align="center">
  <a href="http://nestjs.com/" target="blank"><img src="https://nestjs.com/img/logo-small.svg" width="120" alt="Nest Logo" /></a>
</p>

<h1 align="center">NestJS Production Template</h1>

<p align="center">
  <strong>Enterprise-Grade NestJS Template with Comprehensive Development Standards, Skills, and Practices</strong>
</p>

<p align="center">
  <a href="https://www.npmjs.com/~nestjscore" target="_blank"><img src="https://img.shields.io/npm/v/@nestjs/core.svg" alt="NPM Version" /></a>
  <a href="https://www.npmjs.com/~nestjscore" target="_blank"><img src="https://img.shields.io/npm/l/@nestjs/core.svg" alt="Package License" /></a>
  <a href="https://nodejs.org/" target="_blank"><img src="https://img.shields.io/badge/node-%3E%3D18.0.0-brightgreen" alt="Node Version" /></a>
  <a href="https://www.typescriptlang.org/" target="_blank"><img src="https://img.shields.io/badge/TypeScript-strict-blue" alt="TypeScript" /></a>
</p>

---

## 📋 Description

This is a **production-ready NestJS template** designed to accelerate enterprise application development with built-in best practices, comprehensive development skills, and automated quality gates. It provides:

- **Modular Architecture**: Feature-based module organization with clear separation of concerns
- **Production Standards**: ESLint, Prettier, TypeScript strict mode, and pre-commit hooks
- **Comprehensive Testing**: Unit tests (Jest) with ≥70% coverage and E2E tests (SuperTest)
- **Security First**: Authentication (JWT), authorization (RBAC/ABAC), input validation, and OWASP compliance
- **Performance Optimized**: Redis caching, Bull job queues, database optimization, and query acceleration
- **Error Handling & Logging**: Global exception filters, structured logging (Pino), and custom error classes
- **API Documentation**: Swagger/OpenAPI decorators with automatic documentation generation
- **CI/CD Ready**: GitHub Actions workflows, branch protection, and deployment automation
- **Developer Experience**: Cursor AI skills for intelligent development guidance, checklists, and patterns

---

## 🚀 Quick Start

### Installation

```bash
$ npm install
```

### Environment Setup

```bash
# Copy environment template
$ cp .env.example .env

# Configure your local environment
$ nano .env
```

### Running the Application

```bash
# Development
$ npm run start

# Watch mode (auto-reload)
$ npm run start:dev

# Production
$ npm run start:prod
```

### Running Tests

```bash
# Unit tests
$ npm run test

# Unit tests with coverage
$ npm run test:cov

# E2E tests
$ npm run test:e2e

# Watch mode (development)
$ npm run test:watch
```

### Code Quality

```bash
# Linting
$ npm run lint
$ npm run lint:fix

# Formatting
$ npm run format
$ npm run format:check

# Type checking
$ npm run type-check

# Pre-commit checks
$ git commit  # Automatically runs husky hooks
```

---

## 📁 Project Structure

```
src/
├── common/                 # Shared utilities, decorators, filters, guards
│   ├── decorators/
│   ├── filters/
│   ├── guards/
│   ├── interceptors/
│   └── pipes/
├── config/                 # Centralized configuration
├── <feature>/              # Feature modules (users, products, orders, etc.)
│   ├── <feature>.module.ts
│   ├── <feature>.controller.ts
│   ├── <feature>.controller.spec.ts
│   ├── <feature>.service.ts
│   ├── <feature>.service.spec.ts
│   ├── <feature>.repository.ts
│   ├── dto/
│   │   ├── create-<feature>.dto.ts
│   │   └── update-<feature>.dto.ts
│   └── entities/
│       └── <feature>.entity.ts
└── main.ts                 # Application entry point

test/
├── e2e/
│   └── <feature>.e2e-spec.ts
├── fixtures/               # Test data and seeds
└── jest-e2e.json          # E2E test configuration

.cursor/
├── rules/
│   └── nestjs-standards.mdc    # Project-wide standards
└── skills/                      # AI-powered development guidance
    ├── code-conventions/        # Naming, structure, consistency
    ├── architecture/            # Layered design, patterns, boundaries
    ├── unit-testing/            # Jest, mocking, coverage
    ├── e2e-testing/             # SuperTest, integration testing
    ├── error-handling/          # Exceptions, logging, monitoring
    ├── security/                # Authentication, authorization, OWASP
    ├── configuration/           # Environment variables, secrets
    ├── documentation/           # Swagger, OpenAPI, API contracts
    ├── performance/             # Caching, queues, database optimization
    ├── repo-hygiene/            # Linting, CI/CD, branch protection
    └── feature-delivery/        # Complete feature workflow orchestration
```

---

## 🎯 Core Features

### ✅ Modular Design
- Feature-based module organization
- Clear layer separation (Controllers → Services → Repositories)
- Shared utilities in `common/` folder
- Explicit exports and dependency injection

### ✅ Layered Architecture
- **Presentation Layer**: Controllers, DTOs, Guards
- **Application Layer**: Services, Business Logic
- **Infrastructure Layer**: Repositories, Database Access
- **Domain Layer**: Entities, Business Rules

### ✅ Testing Excellence
- **Unit Tests**: Comprehensive service/controller tests with mocking
- **E2E Tests**: Full API workflow validation with SuperTest
- **Coverage Thresholds**: Enforced ≥70% coverage across all metrics
- **Test Isolation**: Transaction rollback, database cleanup, no test interdependencies

### ✅ Security
- **Authentication**: JWT tokens, refresh tokens, session management
- **Authorization**: Role-Based Access Control (RBAC), Resource-Based Access Control, Attribute-Based Access Control (ABAC)
- **Input Validation**: `class-validator` with automatic DTOs
- **OWASP Protection**: SQL injection prevention, XSS protection, CSRF tokens, secure headers
- **Secrets Management**: Environment-based secrets, never hardcoded credentials

### ✅ Performance & Scalability
- **Redis Caching**: Multi-strategy caching (write-through, write-back, refresh-ahead)
- **Bull Job Queues**: Async processing, retries, dead letter queues
- **Database Optimization**: Indexes, query optimization, N+1 prevention, pagination
- **Connection Pooling**: Optimized database connections

### ✅ Error Handling & Logging
- **Global Exception Filters**: Centralized error handling, consistent responses
- **Structured Logging**: Pino-based logging with context and levels
- **Custom Error Classes**: Domain-specific exceptions with proper HTTP status
- **Error Monitoring**: Health checks, performance monitoring, alerting

### ✅ API Documentation
- **Swagger/OpenAPI**: Automatic API documentation generation
- **DTO as Contracts**: DTOs define API contracts and validation rules
- **Decorators**: `@ApiProperty`, `@ApiResponse`, authentication decorators
- **Spec Synchronization**: Keep OpenAPI specs aligned with code

### ✅ CI/CD Pipeline
- **GitHub Actions**: Automated testing, linting, security audits
- **Pre-Commit Hooks**: Local quality checks before commits
- **Branch Protection**: Main branch always deployable
- **Status Checks**: All tests and quality gates required before merge
- **Deployment Automation**: Safe deployment to production

### ✅ Developer Experience
- **Cursor AI Skills**: Contextual development guidance and patterns
- **Comprehensive Checklists**: Feature delivery workflow with quality gates
- **Best Practices**: Enforced through linting, pre-commit hooks, CI/CD
- **Clear Documentation**: Architecture, security, performance guidance

---

## 🎓 Development Workflow

### Using Cursor AI Skills

This template includes **11 comprehensive Cursor AI skills** that guide you through every aspect of development:

#### **1. Code Conventions Skill** 
Enforces naming, folder structure, DTO usage, and linting rules
- When: Creating new modules, services, controllers
- Reference: `.cursor/skills/code-conventions/`

#### **2. Architecture Skill**
Validates layered architecture, prevents mixing concerns, encourages modular design
- When: Designing new features, reviewing code
- Reference: `.cursor/skills/architecture/`

#### **3. Unit Testing Skill**
Comprehensive Jest testing with mocking patterns and coverage thresholds
- When: Writing service/controller tests
- Reference: `.cursor/skills/unit-testing/`

#### **4. E2E Testing Skill**
SuperTest-based integration testing with environment setup
- When: Testing complete API workflows
- Reference: `.cursor/skills/e2e-testing/`

#### **5. Error Handling & Logging Skill**
Global exception filters, structured logging, custom error classes
- When: Implementing error handling and logging
- Reference: `.cursor/skills/error-handling/`

#### **6. Security Skill**
Authentication guards, authorization checks, input validation, OWASP compliance
- When: Implementing security features
- Reference: `.cursor/skills/security/`

#### **7. Configuration & Environment Skill**
Centralized configuration, environment-based settings, secret management
- When: Adding new configuration or environment variables
- Reference: `.cursor/skills/configuration/`

#### **8. Documentation Skill**
Swagger decorators, API contracts, OpenAPI spec synchronization
- When: Creating API endpoints
- Reference: `.cursor/skills/documentation/`

#### **9. Performance & Scalability Skill**
Redis caching, Bull job queues, database optimization, query acceleration
- When: Optimizing heavy operations
- Reference: `.cursor/skills/performance/`

#### **10. Repository Hygiene Skill**
Linting, formatting, pre-commit hooks, CI/CD pipelines, branch protection
- When: Setting up quality gates
- Reference: `.cursor/skills/repo-hygiene/`

#### **11. Feature Delivery Skill** ⭐
Orchestrates all skills through a complete feature delivery workflow (planning → development → testing → deployment)
- When: Starting a new user story or enhancement
- Reference: `.cursor/skills/feature-delivery/`

### Complete Feature Delivery Process

```mermaid
Planning & Design
    ↓
Implementation (using Code Conventions, Architecture)
    ↓
Unit Testing (using Unit Testing)
    ↓
E2E Testing (using E2E Testing)
    ↓
Code Quality (using Repository Hygiene)
    ↓
CI/CD Pipeline (using Repository Hygiene, GitHub Actions)
    ↓
Performance Review (using Performance Skill)
    ↓
Deployment (using Configuration, Repository Hygiene)
    ↓
Documentation & Monitoring (using Documentation, Error Handling)
```

---

## 📚 Standards & Best Practices

### Project Rule: NestJS Standards

The project includes a comprehensive `.cursor/rules/nestjs-standards.mdc` that enforces:

- **Modular Design**: Features in separate modules, shared code in `common/`
- **Naming Conventions**: Consistent file and class naming
- **Layer Separation**: Controllers, Services, Repositories clearly separated
- **Testing**: Colocated unit tests (`*.spec.ts`), E2E tests in `/test`
- **Configuration**: Centralized in `/src/config`
- **Error Handling**: Global exception filters, custom error classes
- **Logging**: Structured logging with consistent levels
- **Security**: Validation, guards, sanitization
- **Performance**: Caching, queues, optimized queries
- **Documentation**: Swagger decorators on all endpoints
- **Repository Hygiene**: Linting, formatting, CI/CD checks

---

## 🔧 Configuration

### Environment Variables

Create `.env` file from `.env.example`:

```env
# Node
NODE_ENV=development
PORT=3000

# Database
DATABASE_URL=postgresql://user:password@localhost:5432/dbname

# JWT
JWT_SECRET=your-secret-key
JWT_EXPIRATION=3600

# Redis
REDIS_HOST=localhost
REDIS_PORT=6379

# Email Service
MAIL_HOST=smtp.example.com
MAIL_USER=user@example.com
MAIL_PASS=password

# External Services
STRIPE_API_KEY=sk_test_...
```

See `.env.example` for all available configuration options.

---

## 🧪 Testing Strategy

### Unit Tests

```bash
# Run all unit tests
$ npm run test

# Watch mode
$ npm run test:watch

# Generate coverage report
$ npm run test:cov

# Coverage thresholds enforced:
# - Branches: 70%
# - Functions: 70%
# - Lines: 70%
# - Statements: 70%
```

### E2E Tests

```bash
# Run E2E tests (requires running server)
$ npm run test:e2e

# E2E tests validate:
# - API contracts
# - Database interactions
# - Complete workflows
# - Error scenarios
# - Authorization checks
```

---

## 🚢 Deployment

### Pre-Deployment Checklist

- [ ] All tests passing locally
- [ ] Coverage ≥70%
- [ ] No linting errors
- [ ] Code review approved
- [ ] All CI/CD checks green
- [ ] Environment variables configured
- [ ] Database migrations applied
- [ ] Secrets configured in CI/CD

### Production Deployment

```bash
# Build for production
$ npm run build

# Run production server
$ npm run start:prod
```

See [NestJS Deployment Documentation](https://docs.nestjs.com/deployment) for detailed deployment strategies.

---

## 📖 Documentation

### Architecture Decisions

See `.cursor/skills/architecture/` for:
- Layered architecture patterns
- Module design principles
- Architectural violations and fixes
- When to use caching vs. queues

### Security Guidelines

See `.cursor/skills/security/` for:
- Authentication implementation (JWT, OAuth, sessions)
- Authorization patterns (RBAC, ABAC, ACL)
- OWASP Top 10 protection
- Input validation and sanitization

### Performance Optimization

See `.cursor/skills/performance/` for:
- Caching strategies (Redis, in-memory)
- Job queues (Bull, async processing)
- Database optimization (indexes, queries, pagination)
- Query performance monitoring

### Testing Practices

See `.cursor/skills/unit-testing/` and `.cursor/skills/e2e-testing/` for:
- Mocking complex dependencies
- Coverage improvement strategies
- Test organization and naming
- E2E workflow testing patterns

---

## 🎯 Common Tasks

### Creating a New Feature

1. Follow the **Feature Delivery Skill** (`.cursor/skills/feature-delivery/SKILL.md`)
2. Use the **Skill Matrix** to identify which skills to reference
3. Use **Code Conventions** for module structure
4. Follow **Architecture** patterns
5. Write tests using **Unit Testing** and **E2E Testing** skills
6. Document with **Documentation** skill
7. Optimize with **Performance** skill
8. Ensure **Repository Hygiene**

### Implementing Authentication

1. Start with **Security Skill** - Authentication section
2. Add JWT configuration using **Configuration Skill**
3. Implement guards and strategies
4. Write tests with **Unit Testing** and **E2E Testing**
5. Document endpoints with **Documentation** skill

### Optimizing Performance

1. Identify bottleneck using **Performance** skill
2. Add database indexes using **Performance - Database** skill
3. Implement caching using **Performance - Caching** skill
4. Use queues for heavy operations using **Performance - Queuing** skill
5. Monitor with **Error Handling - Monitoring** skill

### Debugging Failed Tests

1. Check **Unit Testing - Coverage** skill for coverage gaps
2. Review **E2E Testing - Database** skill for test isolation issues
3. Use **Repository Hygiene - Troubleshooting** for CI/CD issues

---

## 📦 Dependencies

### Core
- `@nestjs/core` - NestJS framework
- `@nestjs/common` - Common utilities
- `reflect-metadata` - Decorator support
- `typescript` - TypeScript compiler

### Validation & Security
- `class-validator` - Input validation
- `class-transformer` - DTO transformation
- `@nestjs/jwt` - JWT authentication
- `@nestjs/passport` - Authentication strategies
- `passport-jwt` - JWT strategy
- `bcrypt` - Password hashing
- `helmet` - Security headers

### Database & ORM
- `typeorm` - ORM
- `pg` - PostgreSQL driver

### Caching & Queues
- `@nestjs/cache-manager` - Caching module
- `cache-manager` - Cache manager
- `@nestjs/bull` - Job queues
- `bull` - Bull queue

### Logging
- `@nestjs/pino` - Pino integration
- `pino` - Structured logging

### Documentation
- `@nestjs/swagger` - Swagger integration
- `swagger-ui-express` - Swagger UI

### Testing
- `jest` - Testing framework
- `@nestjs/testing` - NestJS testing utilities
- `supertest` - HTTP testing

### Development
- `eslint` - Code linting
- `prettier` - Code formatting
- `husky` - Git hooks
- `lint-staged` - Staged file linting

---

## 📚 Resources

### Documentation
- [NestJS Official Documentation](https://docs.nestjs.com)
- [NestJS GitHub Repository](https://github.com/nestjs/nest)
- [TypeScript Documentation](https://www.typescriptlang.org/docs/)

### Learning
- [NestJS Official Courses](https://courses.nestjs.com/)
- [NestJS Discord Community](https://discord.gg/G7Qnnhy)

### Tools
- [NestJS Devtools](https://devtools.nestjs.com) - Visualize application graph
- [NestJS Enterprise Support](https://enterprise.nestjs.com) - Professional support

---

## 🤝 Contributing

When contributing to this template:

1. Follow the **Code Conventions** skill for naming and structure
2. Follow the **Architecture** skill for design patterns
3. Write tests meeting the **Unit Testing** and **E2E Testing** requirements
4. Ensure **Repository Hygiene** standards (linting, formatting, pre-commit)
5. Update **Documentation** for API changes
6. Check **Security** for any security implications

---

## 📄 License

This project is [MIT licensed](https://github.com/nestjs/nest/blob/master/LICENSE).

---

## 🙏 Acknowledgments

Built on the excellent [NestJS Framework](https://nestjs.com/) by [Kamil Myśliwiec](https://twitter.com/kammysliwiec) and the NestJS community.

This template adds comprehensive enterprise standards, AI-powered Cursor skills, and production-grade practices to accelerate team development while maintaining consistency, quality, and security.

---

## 📞 Support

For issues, questions, or contributions:
- Open an issue in the repository
- Consult the relevant **Cursor Skill** documentation (`.cursor/skills/`)
- Check the **Feature Delivery Skill** for workflow guidance
- Review **Repository Hygiene** troubleshooting guide

---

**Happy Coding! 🚀**
