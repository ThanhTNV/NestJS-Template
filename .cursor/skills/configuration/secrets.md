# Secrets Management

This document covers secure handling of secrets and sensitive configuration.

## What is a Secret?

Secrets are sensitive values that must be protected:
- Passwords (database, API keys, etc.)
- API keys (3rd party services)
- Encryption keys (JWT, session, etc.)
- Private certificates
- OAuth tokens
- Credit card data
- PII (personally identifiable information)

**Rule**: Any value you wouldn't want in a public repository is a secret.

## Storage Strategies

### Local Development

Use `.env.local` (git-ignored):

```bash
# .env.local
DATABASE_PASSWORD=local_development_password_not_a_secret
JWT_SECRET=dev-secret-key-unsafe-change-me
STRIPE_SECRET_KEY=sk_test_local_testing_key
```

**Security**: Low (local machine only)

### Staging/Production

Use external secret managers:

| Service | Use Case | Provider |
|---------|----------|----------|
| AWS Secrets Manager | AWS environments | Amazon |
| HashiCorp Vault | On-premise, multi-cloud | HashiCorp |
| Azure Key Vault | Azure environments | Microsoft |
| Google Secret Manager | GCP environments | Google |
| 1Password | Team-based secrets | 1Password |
| Doppler | CI/CD integration | Doppler |

## AWS Secrets Manager Integration

### Store Secret

```bash
# Store password in AWS Secrets Manager
aws secretsmanager create-secret \
  --name prod/database/password \
  --secret-string "your-strong-password" \
  --region us-east-1
```

### Retrieve Secret in NestJS

```typescript
// src/config/aws-secrets.ts
import * as AWS from 'aws-sdk';

const secretsManager = new AWS.SecretsManager({
  region: process.env.AWS_REGION || 'us-east-1',
});

export async function getSecret(secretName: string): Promise<string> {
  try {
    const data = await secretsManager.getSecretValue({
      SecretId: secretName,
    }).promise();

    if ('SecretString' in data) {
      return data.SecretString;
    } else {
      const buff = Buffer.from(data.SecretBinary as string, 'base64');
      return buff.toString('ascii');
    }
  } catch (error) {
    console.error(`Failed to retrieve secret: ${secretName}`, error);
    throw error;
  }
}

// Usage in config
export default registerAs('database', async (): Promise<IDatabaseConfig> => {
  const password = await getSecret('prod/database/password');

  return {
    host: process.env.DATABASE_HOST,
    port: parseInt(process.env.DATABASE_PORT),
    username: process.env.DATABASE_USERNAME,
    password, // From AWS
    database: process.env.DATABASE_NAME,
  };
});
```

### Docker with AWS Secrets

```dockerfile
FROM node:18-alpine

WORKDIR /app

COPY package*.json ./
RUN npm ci

COPY . .
RUN npm run build

# At runtime, AWS SDK will fetch secrets
CMD ["node", "dist/main.js"]
```

## HashiCorp Vault Integration

### Store Secret

```bash
vault kv put secret/myapp/db \
  host=prod-db.example.com \
  username=app_user \
  password=strong-password
```

### Retrieve in NestJS

```typescript
// src/config/vault.service.ts
import * as VaultClient from 'node-vault';

const vault = new VaultClient({
  endpoint: process.env.VAULT_ADDR,
  token: process.env.VAULT_TOKEN,
});

export async function getVaultSecret(path: string): Promise<Record<string, any>> {
  try {
    const result = await vault.read(`secret/data/${path}`);
    return result.data.data;
  } catch (error) {
    console.error(`Failed to read from Vault: ${path}`, error);
    throw error;
  }
}

// Usage
export default registerAs('database', async (): Promise<IDatabaseConfig> => {
  const secrets = await getVaultSecret('myapp/db');

  return {
    host: secrets.host,
    port: 5432,
    username: secrets.username,
    password: secrets.password,
    database: process.env.DATABASE_NAME,
  };
});
```

## CI/CD Secrets

### GitHub Actions

```yaml
# .github/workflows/deploy.yml
env:
  DATABASE_HOST: ${{ secrets.DATABASE_HOST }}
  DATABASE_USERNAME: ${{ secrets.DATABASE_USERNAME }}
  DATABASE_PASSWORD: ${{ secrets.DATABASE_PASSWORD }}
  JWT_SECRET: ${{ secrets.JWT_SECRET }}

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
      - name: Deploy
        run: npm run deploy
```

### GitLab CI

```yaml
# .gitlab-ci.yml
variables:
  DATABASE_HOST: $DATABASE_HOST
  DATABASE_PASSWORD: $DATABASE_PASSWORD  # From project/group secrets
  JWT_SECRET: $JWT_SECRET

deploy:
  stage: deploy
  script:
    - npm run build
    - npm run deploy
```

## Rotation Strategy

### Schedule Rotation

```bash
# Rotate secrets every 30 days
0 0 * * * rotate-secrets.sh
```

### Rotation Script

```bash
#!/bin/bash
# scripts/rotate-secrets.sh

SECRETS=(
  "prod/database/password"
  "prod/jwt/secret"
  "prod/redis/password"
)

for secret in "${SECRETS[@]}"; do
  NEW_PASSWORD=$(openssl rand -base64 32)
  
  aws secretsmanager put-secret-value \
    --secret-id "$secret" \
    --secret-string "$NEW_PASSWORD"
  
  echo "Rotated: $secret"
done

# Notify team
echo "Secrets rotated. Update services as needed."
```

## Audit & Monitoring

### Log Secret Access

```typescript
// src/common/middleware/secret-access.middleware.ts
@Injectable()
export class SecretAccessMiddleware implements NestMiddleware {
  use(req: Request, res: Response, next: NextFunction) {
    // Log access attempts (without logging secret values!)
    this.logger.log({
      event: 'secret_access',
      user: req.user?.id,
      endpoint: req.path,
      timestamp: new Date(),
    });

    next();
  }
}
```

### Alert on Suspicious Activity

```typescript
// Detect unusual secret access patterns
if (consecutiveFailures > 5) {
  alertOncall('Suspicious secret access pattern detected');
}
```

## Never Do This ❌

### ❌ Hardcoded Secrets

```typescript
// NEVER!
const DB_PASSWORD = 'mySecretPassword123';
const API_KEY = 'sk_live_abc123xyz';

const config = {
  db: { password: 'database_password' },
  api: { key: 'my-secret-api-key' },
};
```

### ❌ Secrets in Git History

```bash
# BAD - Already in history!
git add .env
git commit -m "Add environment config"
git push

# Clean up (if discovered)
git log --all --full-history -- .env  # Find commits
git filter-branch --prune-empty --tree-filter 'rm -rf .env' -- --all
git push origin --force --all
```

### ❌ Secrets in Logs

```typescript
// BAD
this.logger.log(`User login with password: ${password}`);
this.logger.log(`API Response:`, apiResponse); // May contain secrets

// GOOD
this.logger.log(`User login successful`);
this.logger.log(`API Response:`, sanitize(apiResponse));
```

### ❌ Secrets in Error Messages

```typescript
// BAD
throw new Error(`Database error: ${connectionString}`);

// GOOD
throw new Error('Database connection failed');
this.logger.error('Database error', { error, sanitized: true });
```

### ❌ Default/Weak Secrets

```bash
# BAD - Default in code
API_KEY=abc123
DATABASE_PASSWORD=password

# GOOD - Strong, unique per environment
API_KEY=$(openssl rand -base64 32)
DATABASE_PASSWORD=$(openssl rand -base64 32)
```

## Security Checklist

- [ ] No secrets committed to git
- [ ] `.env` files in `.gitignore`
- [ ] `.env.example` committed (template only)
- [ ] Strong secrets (>32 characters, mixed case)
- [ ] Unique secrets per environment
- [ ] Secrets rotated regularly (monthly)
- [ ] Secrets audited (access logs)
- [ ] Secrets encrypted at rest
- [ ] Secrets encrypted in transit (HTTPS)
- [ ] No secrets in logs
- [ ] No secrets in error messages
- [ ] Access restricted to authorized personnel
- [ ] Revocation process documented
- [ ] Backup secrets secure

## Incident Response

### If Secret is Compromised

1. **Rotate immediately**
   ```bash
   aws secretsmanager rotate-secret --secret-id compromised-secret
   ```

2. **Revoke access**
   ```bash
   # Revoke API key, disable user, etc.
   ```

3. **Audit access logs**
   ```bash
   # Check who accessed and what they did
   ```

4. **Notify stakeholders**
   - Security team
   - Affected customers
   - Incident response team

5. **Update credentials everywhere**
   - Database connections
   - API clients
   - External services
   - CI/CD pipelines

6. **Document incident**
   - Timeline
   - Impact
   - Remediation
   - Prevention

## Tools for Secret Detection

### Prevent Accidental Commits

```bash
# Install pre-commit hook
npm install pre-commit --save-dev

# Add to package.json
"pre-commit": [
  "detect-secrets scan",
  "eslint",
  "prettier"
]
```

### Scan for Secrets

```bash
# Using Trufflehog
docker run -v "$PWD:/path" trufflesecurity/trufflehog \
  filesystem /path --json

# Using detect-secrets
detect-secrets scan
```
