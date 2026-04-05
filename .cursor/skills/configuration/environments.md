# Environment Files & Examples

This document provides detailed environment configuration templates for each deployment stage.

## File Priority & Loading Order

NestJS Config loads environment files in this order (first found wins):

1. `.env.${NODE_ENV}.local` - Local machine overrides (git-ignored)
2. `.env.${NODE_ENV}` - Environment-specific
3. `.env.local` - Local development (git-ignored)
4. `.env` - Default (rarely used, normally git-ignored)

**Example**: If `NODE_ENV=staging`, loads: `.env.staging.local` → `.env.staging` → `.env.local` → `.env`

## .env.example (Git-Tracked Template)

This file documents all required and optional variables. Keep it up-to-date!

```bash
# ============================================
# Application Configuration
# ============================================
NODE_ENV=development
APP_NAME=MyApp
APP_PORT=3000
APP_URL=http://localhost:3000
LOG_LEVEL=debug

# ============================================
# Database Configuration
# ============================================
DATABASE_HOST=localhost
DATABASE_PORT=5432
DATABASE_USERNAME=user
DATABASE_PASSWORD=password
DATABASE_NAME=myapp_dev
DATABASE_SSL=false
DATABASE_LOGGING=false

# ============================================
# JWT Configuration
# ============================================
JWT_SECRET=dev-secret-never-use-in-production
JWT_EXPIRATION=24h
JWT_REFRESH_SECRET=dev-refresh-secret
JWT_REFRESH_EXPIRATION=7d

# ============================================
# Cache Configuration
# ============================================
CACHE_TTL=3600
REDIS_HOST=localhost
REDIS_PORT=6379

# ============================================
# External Services (Examples)
# ============================================
# STRIPE_SECRET_KEY=sk_test_...
# SENDGRID_API_KEY=SG...
# AWS_ACCESS_KEY_ID=AKIA...
# AWS_SECRET_ACCESS_KEY=...
```

## Development (.env.local)

For local machine development. Never commit to git.

```bash
# ============================================
# Application
# ============================================
NODE_ENV=development
APP_NAME=MyApp-Local
APP_PORT=3000
APP_URL=http://localhost:3000
LOG_LEVEL=debug

# ============================================
# Database (Local)
# ============================================
# Typically running in Docker or local install
DATABASE_HOST=localhost
DATABASE_PORT=5432
DATABASE_USERNAME=dev_user
DATABASE_PASSWORD=dev_pass_123
DATABASE_NAME=myapp_dev
DATABASE_SSL=false
DATABASE_LOGGING=true  # Log SQL in development

# ============================================
# JWT (Development Keys - Not Secret!)
# ============================================
JWT_SECRET=local-dev-secret-key-unsafe-change-me
JWT_EXPIRATION=24h
JWT_REFRESH_SECRET=local-dev-refresh-key
JWT_REFRESH_EXPIRATION=7d

# ============================================
# Cache (Local Redis)
# ============================================
CACHE_TTL=3600
REDIS_HOST=localhost
REDIS_PORT=6379

# ============================================
# Development Tools
# ============================================
DEBUG=myapp:*
TYPEORM_LOGGING=true
PROFILING_ENABLED=true
```

## Staging (.env.staging)

For staging environment. Use strong values, load secrets from CI/CD system.

```bash
# ============================================
# Application
# ============================================
NODE_ENV=staging
APP_NAME=MyApp-Staging
APP_PORT=3000
APP_URL=https://staging-api.example.com
LOG_LEVEL=info

# ============================================
# Database (Staging - RDS/Managed)
# ============================================
DATABASE_HOST=staging-db-instance.us-east-1.rds.amazonaws.com
DATABASE_PORT=5432
DATABASE_USERNAME=stg_admin_user
DATABASE_PASSWORD=${STAGING_DB_PASSWORD}  # From CI/CD secrets
DATABASE_NAME=myapp_staging
DATABASE_SSL=true
DATABASE_LOGGING=false

# ============================================
# JWT (Staging - Staging Keys)
# ============================================
JWT_SECRET=${STAGING_JWT_SECRET}           # From CI/CD secrets
JWT_EXPIRATION=24h
JWT_REFRESH_SECRET=${STAGING_JWT_REFRESH}  # From CI/CD secrets
JWT_REFRESH_EXPIRATION=7d

# ============================================
# Cache (Staging - ElastiCache)
# ============================================
CACHE_TTL=3600
REDIS_HOST=staging-redis-instance.us-east-1.cache.amazonaws.com
REDIS_PORT=6379
REDIS_PASSWORD=${STAGING_REDIS_PASSWORD}   # From CI/CD secrets

# ============================================
# External Services (Staging Credentials)
# ============================================
STRIPE_SECRET_KEY=${STAGING_STRIPE_SECRET}
SENDGRID_API_KEY=${STAGING_SENDGRID_KEY}
AWS_REGION=us-east-1
AWS_ACCESS_KEY_ID=${STAGING_AWS_ACCESS_KEY}
AWS_SECRET_ACCESS_KEY=${STAGING_AWS_SECRET}

# ============================================
# Monitoring & Observability
# ============================================
SENTRY_DSN=${STAGING_SENTRY_DSN}
NEW_RELIC_APP_NAME=MyApp-Staging
NEW_RELIC_LICENSE_KEY=${STAGING_NEW_RELIC_KEY}

# ============================================
# Feature Flags
# ============================================
FEATURE_NEW_CHECKOUT=true
FEATURE_EMAIL_VERIFICATION=false
```

## Production (.env.production)

For production. Use secret manager, never commit secrets!

```bash
# ============================================
# Application
# ============================================
NODE_ENV=production
APP_NAME=MyApp
APP_PORT=3000
APP_URL=https://api.example.com
LOG_LEVEL=warn  # Minimal logging in production

# ============================================
# Database (Production - RDS/Managed)
# ============================================
DATABASE_HOST=prod-db-instance.us-east-1.rds.amazonaws.com
DATABASE_PORT=5432
DATABASE_USERNAME=prod_app_user
DATABASE_PASSWORD=${DATABASE_PASSWORD}     # From AWS Secrets Manager
DATABASE_NAME=myapp_prod
DATABASE_SSL=true
DATABASE_LOGGING=false

# ============================================
# JWT (Production - Strong Keys!)
# ============================================
JWT_SECRET=${JWT_SECRET}                   # From AWS Secrets Manager
JWT_EXPIRATION=12h  # Shorter in production
JWT_REFRESH_SECRET=${JWT_REFRESH_SECRET}   # From AWS Secrets Manager
JWT_REFRESH_EXPIRATION=7d

# ============================================
# Cache (Production - ElastiCache Cluster)
# ============================================
CACHE_TTL=3600
REDIS_CLUSTER_ENDPOINTS=${REDIS_ENDPOINTS}  # Multiple endpoints
REDIS_PASSWORD=${REDIS_PASSWORD}           # From AWS Secrets Manager

# ============================================
# External Services (Production Credentials)
# ============================================
STRIPE_SECRET_KEY=${STRIPE_SECRET}         # From AWS Secrets Manager
SENDGRID_API_KEY=${SENDGRID_API_KEY}       # From AWS Secrets Manager
AWS_REGION=us-east-1
AWS_ACCESS_KEY_ID=${AWS_ACCESS_KEY}        # From AWS Secrets Manager
AWS_SECRET_ACCESS_KEY=${AWS_SECRET_KEY}    # From AWS Secrets Manager

# ============================================
# Monitoring & Observability (Production)
# ============================================
SENTRY_DSN=${SENTRY_DSN}                   # From AWS Secrets Manager
SENTRY_ENVIRONMENT=production
NEW_RELIC_APP_NAME=MyApp-Production
NEW_RELIC_LICENSE_KEY=${NEW_RELIC_LICENSE} # From AWS Secrets Manager
DATADOG_API_KEY=${DATADOG_API_KEY}        # From AWS Secrets Manager

# ============================================
# Performance & Scaling
# ============================================
NODEJS_MAX_HTTP_CLIENTS=1024
DATABASE_POOL_SIZE=20
CACHE_INVALIDATION_TTL=300

# ============================================
# Feature Flags (Production)
# ============================================
FEATURE_NEW_CHECKOUT=true
FEATURE_EMAIL_VERIFICATION=true
FEATURE_ADVANCED_REPORTING=true
```

## Loading from CI/CD

### GitHub Actions Example

```yaml
# .github/workflows/deploy-staging.yml
name: Deploy Staging

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2

      - name: Set environment variables
        run: |
          echo "STAGING_DB_PASSWORD=${{ secrets.STAGING_DB_PASSWORD }}" >> .env.staging
          echo "STAGING_JWT_SECRET=${{ secrets.STAGING_JWT_SECRET }}" >> .env.staging
          echo "STAGING_REDIS_PASSWORD=${{ secrets.STAGING_REDIS_PASSWORD }}" >> .env.staging

      - name: Build and deploy
        run: npm run build && npm run deploy:staging
```

### Docker Environment Variables

```dockerfile
# Dockerfile
FROM node:18-alpine

WORKDIR /app

COPY package*.json ./
RUN npm ci

COPY . .
RUN npm run build

# Pass environment at runtime
CMD ["node", "dist/main.js"]
```

```bash
# Docker run with environment
docker run -d \
  -e NODE_ENV=production \
  -e DATABASE_PASSWORD=$DB_PASSWORD \
  -e JWT_SECRET=$JWT_SECRET \
  -e REDIS_PASSWORD=$REDIS_PASSWORD \
  myapp:latest
```

### Kubernetes ConfigMap & Secrets

```yaml
# config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  NODE_ENV: "production"
  APP_NAME: "MyApp"
  LOG_LEVEL: "warn"
---
apiVersion: v1
kind: Secret
metadata:
  name: app-secrets
type: Opaque
stringData:
  DATABASE_PASSWORD: <base64-encoded>
  JWT_SECRET: <base64-encoded>
  REDIS_PASSWORD: <base64-encoded>
---
apiVersion: v1
kind: Pod
metadata:
  name: app-pod
spec:
  containers:
  - name: app
    image: myapp:latest
    envFrom:
    - configMapRef:
        name: app-config
    - secretRef:
        name: app-secrets
```

## Local Development Setup

### Setup Script

```bash
#!/bin/bash
# scripts/setup-env.sh

# Copy template
cp .env.example .env.local

# Prompt for local values
read -p "Enter local database password: " DB_PASS
sed -i "s/DATABASE_PASSWORD=.*/DATABASE_PASSWORD=$DB_PASS/" .env.local

read -p "Enter local Redis password (optional): " REDIS_PASS
sed -i "s/REDIS_PASSWORD=.*/REDIS_PASSWORD=$REDIS_PASS/" .env.local

echo "✓ .env.local created"
echo "✓ Please review and edit as needed"
```

### Makefile for Environment Management

```makefile
.PHONY: env-setup env-check env-list

env-setup:
	@if [ ! -f .env.local ]; then \
		cp .env.example .env.local; \
		echo "✓ .env.local created from template"; \
	else \
		echo "✓ .env.local already exists"; \
	fi

env-check:
	@echo "Checking environment variables..."
	@grep -E "^[^#]" .env.local | head -10

env-list:
	@echo "Available environment files:"
	@ls -lh .env* 2>/dev/null || echo "No .env files found"
```

Usage:
```bash
make env-setup  # Create .env.local from template
make env-check  # View current config
make env-list   # List all .env files
```

## .gitignore Configuration

```bash
# .gitignore

# Environment files (secrets - NEVER commit!)
.env
.env.local
.env.*.local
.env.*.private
.env.secret

# Keep template
!.env.example

# Alternative naming patterns
config.local.ts
secrets.json
credentials.json

# IDE environment files
.vscode/launch.json
.idea/runConfigurations/
```

## Best Practices

### DO ✅

- Keep `.env.example` with all possible variables
- Use strong, unique secrets per environment
- Load secrets from external systems (AWS Secrets Manager, Vault)
- Validate all required variables on startup
- Document what each variable does
- Rotate secrets regularly
- Use different keys for dev/staging/prod
- Log configuration load status (without secrets)

### DON'T ❌

- Never commit actual `.env` files
- Never hardcode secrets in code
- Don't use same secrets across environments
- Don't leave development secrets in git history
- Don't use weak or predictable values
- Don't expose secrets in logs
- Don't skip validation
- Don't leave debug flags in production
