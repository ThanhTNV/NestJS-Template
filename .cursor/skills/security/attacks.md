# Attack Vectors & Prevention

This document covers common security attacks and how to prevent them.

## OWASP Top 10 (2021)

### 1. Broken Access Control

**Vulnerability:** Users can access resources they shouldn't.

**Examples:**
```typescript
// ❌ BAD - No authorization check
@Get('/users/:id')
async getUser(@Param('id') id: string) {
  return this.userService.findById(id); // Anyone can access any user!
}

// ✅ GOOD - Verify ownership or roles
@Get('/users/:id')
@UseGuards(JwtAuthGuard)
async getUser(@Param('id') id: string, @Request() req) {
  if (id !== req.user.id && !req.user.roles.includes('admin')) {
    throw new ForbiddenException('Cannot access other users');
  }
  return this.userService.findById(id);
}
```

**Prevention:**
- Use guards on all protected endpoints
- Check user ownership/permissions
- Default deny, explicitly allow
- Implement RBAC

### 2. Cryptographic Failures

**Vulnerability:** Sensitive data exposed due to weak encryption.

**Examples:**
```typescript
// ❌ BAD - Plain text passwords
user.password = 'plaintext123'; // NEVER!

// ❌ BAD - Weak hashing
import crypto from 'crypto';
const hash = crypto.createHash('md5').update(password).digest('hex'); // MD5 is weak!

// ✅ GOOD - bcrypt with strong salt
const hash = await bcrypt.hash(password, 12);
```

**Prevention:**
- Use bcrypt/scrypt for passwords (salt rounds ≥ 10)
- Use strong TLS/SSL (HTTPS everywhere)
- Don't store sensitive data unnecessarily
- Encrypt data at rest
- Rotate encryption keys

### 3. Injection Attacks (SQL, NoSQL, Command)

**Vulnerability:** Untrusted data interpreted as code.

**SQL Injection Examples:**
```typescript
// ❌ BAD - String concatenation
const email = req.body.email; // User input!
const user = await db.query(`SELECT * FROM users WHERE email = '${email}'`);
// Attacker could send: ' OR '1'='1

// ✅ GOOD - Parameterized queries
const user = await db.query(
  'SELECT * FROM users WHERE email = ?',
  [email] // Email treated as data, not code
);

// ✅ GOOD - TypeORM with parameters
const user = await this.userRepository.findOne({
  where: { email },
});
```

**NoSQL Injection Examples:**
```typescript
// ❌ BAD - Direct object manipulation
const user = await User.findOne({ email: req.body.email });
// Attacker sends: { "$ne": null }

// ✅ GOOD - Validate and sanitize
import mongoSanitize from 'mongo-sanitize';
const email = mongoSanitize(req.body.email);
const user = await User.findOne({ email });
```

**Command Injection Examples:**
```typescript
// ❌ BAD - Runtime command execution
import { execSync } from 'child_process';
const result = execSync(`convert ${userInput} image.jpg`);

// ✅ GOOD - Use safe libraries or validate strictly
import sharp from 'sharp';
await sharp(userInput).resize(100, 100).toFile('image.jpg');
```

**Prevention:**
- Use parameterized queries/prepared statements
- Validate input types (DTOs with class-validator)
- Whitelist allowed input characters
- Never concatenate user input into queries
- Use ORMs that prevent injection
- Escape special characters

### 4. Insecure Design

**Vulnerability:** Missing security controls during design phase.

**Examples:**
- No rate limiting on login
- No CSRF protection
- Missing security headers
- No input validation
- Verbose error messages

**Prevention:**
- Design security from the start
- Implement rate limiting
- Use CSRF tokens for state changes
- Add security headers (Helmet)
- Validate all input
- Use generic error messages

### 5. Security Misconfiguration

**Vulnerability:** Default credentials, unnecessary services, unpatched systems.

**Examples:**
```typescript
// ❌ BAD - Hardcoded secrets
const dbPassword = 'admin123'; // NEVER!

// ✅ GOOD - Environment variables
const dbPassword = process.env.DATABASE_PASSWORD;

// ❌ BAD - Exposing debug info in production
if (process.env.NODE_ENV === 'development') {
  return error.stack; // Don't expose in production!
}

// ✅ GOOD - Generic error messages
throw new InternalServerError('An error occurred'); // Details logged privately
```

**Prevention:**
- Store secrets in environment variables
- Disable debug mode in production
- Remove default credentials
- Keep dependencies updated
- Use principle of least privilege
- Enable HTTPS
- Disable unnecessary features

### 6. Vulnerable & Outdated Components

**Vulnerability:** Using libraries with known vulnerabilities.

**Detection & Prevention:**
```bash
# Check for vulnerabilities
npm audit

# Fix known issues
npm audit fix

# Update packages
npm update

# Check specific package
npm ls express
```

### 7. Authentication Failures

**Vulnerability:** Weak or missing authentication.

**Examples:**
```typescript
// ❌ BAD - No password strength check
user.password = await bcrypt.hash(dto.password, 12); // '123'?

// ✅ GOOD - Validate password strength
const strength = this.passwordService.validateStrength(dto.password);
if (!strength.valid) {
  throw new BadRequestException(strength.errors);
}

// ❌ BAD - No rate limiting
@Post('/login')
async login(@Body() dto: LoginDto) {
  // Attacker can brute force!
}

// ✅ GOOD - Rate limiting
@Post('/login')
@UseGuards(RateLimitGuard)
async login(@Body() dto: LoginDto) {
  // Protected from brute force
}
```

**Prevention:**
- Enforce strong passwords
- Implement rate limiting
- Use secure session management
- Implement MFA for sensitive accounts
- Don't expose whether email exists
- Use secure password reset flows

### 8. Software & Data Integrity Failures

**Vulnerability:** Untrusted CI/CD, vulnerable dependencies.

**Examples:**
```typescript
// ❌ BAD - No verification of dependencies
npm install express // Could be compromised!

// ✅ GOOD - Lock file + audit
npm ci // Uses package-lock.json
npm audit
```

**Prevention:**
- Use lock files (package-lock.json)
- Verify package integrity
- Sign commits
- Verify build artifacts
- Keep CI/CD secure
- Regular security audits

### 9. Logging & Monitoring Failures

**Vulnerability:** Not detecting attacks.

**Examples:**
```typescript
// ❌ BAD - No audit logging
async deleteUser(userId: string) {
  await this.userRepository.delete(userId);
  // No record of who deleted what!
}

// ✅ GOOD - Audit logging
async deleteUser(userId: string, deletedBy: string) {
  await this.userRepository.delete(userId);
  this.auditService.log('USER_DELETED', {
    deletedUser: userId,
    deletedBy,
    timestamp: new Date(),
  });
}
```

**Prevention:**
- Log security events (login, access denied, etc.)
- Monitor for anomalies
- Alert on suspicious patterns
- Keep logs secure and immutable
- Review logs regularly

### 10. Server-Side Request Forgery (SSRF)

**Vulnerability:** Server makes requests to unintended destinations.

**Examples:**
```typescript
// ❌ BAD - No validation of URL
const response = await fetch(req.body.url); // Attacker could request internal IPs!

// ✅ GOOD - Validate URL
import { URL } from 'url';
const url = new URL(req.body.url);
const allowed = ['example.com', 'api.example.com'];

if (!allowed.includes(url.hostname)) {
  throw new BadRequestException('URL not allowed');
}

const response = await fetch(req.body.url);
```

**Prevention:**
- Validate and whitelist URLs
- Block access to private IPs (127.0.0.1, 10.0.0.0/8, etc.)
- Disable unused protocols
- Use allowlists, not blocklists

## Other Common Vulnerabilities

### Cross-Site Scripting (XSS)

**Vulnerability:** Injecting malicious scripts into pages.

**Examples:**
```typescript
// ❌ BAD - Rendering unsanitized HTML
res.json({ content: userInput }); // Could contain <script>alert('XSS')</script>

// ✅ GOOD - Sanitize HTML
import DOMPurify from 'dompurify';
const clean = DOMPurify.sanitize(userInput);
res.json({ content: clean });
```

**Prevention:**
- Use frameworks that auto-escape (React, Vue, Angular)
- Sanitize HTML with DOMPurify
- Use Content Security Policy headers
- Validate input types (DTOs)

### Cross-Site Request Forgery (CSRF)

**Vulnerability:** Tricking users into making unwanted requests.

**Example:**
```
1. User logs into bank.com
2. User visits attacker.com (still logged in)
3. attacker.com makes request: POST /bank/transfer
4. Bank executes transfer (user doesn't realize!)
```

**Prevention:**
- Use CSRF tokens
- Use SameSite cookies
- Verify request origin
- Require JWT for API calls (not cookies)

### Sensitive Data Exposure

**Vulnerability:** Exposing private information.

**Examples:**
```typescript
// ❌ BAD - Logging passwords
this.logger.log(`User ${email} password: ${password}`);

// ❌ BAD - Returning sensitive fields
return { ...user, passwordHash: user.password }; // Expose password hash!

// ✅ GOOD - Exclude sensitive fields
@Exclude()
passwordHash: string;

// ✅ GOOD - Don't log sensitive data
this.logger.log(`User login attempt: ${email}`);
```

**Prevention:**
- Never log sensitive data
- Use `@Exclude()` on sensitive fields
- Encrypt data at rest
- Use HTTPS only
- Minimize data collection

### Broken Cryptography

**Vulnerability:** Weak encryption algorithms.

**Examples:**
```typescript
// ❌ BAD - Weak algorithms
import crypto from 'crypto';
const hash = crypto.createHash('md5').update(data).digest();  // MD5 broken!
const hash = crypto.createHash('sha1').update(data).digest(); // SHA1 weak!

// ✅ GOOD - Strong algorithms
const hash = await bcrypt.hash(data, 12);      // For passwords
const hash = crypto.createHash('sha256');       // For general hashing
```

**Prevention:**
- Use bcrypt for passwords
- Use strong hashing (SHA-256+)
- Use TLS 1.2+ for transport
- Rotate keys regularly

## Security Checklist

### Before Deployment

- [ ] All secrets in environment variables
- [ ] HTTPS enabled
- [ ] Security headers configured (Helmet)
- [ ] Rate limiting enabled
- [ ] Input validation on all endpoints
- [ ] Authentication on protected endpoints
- [ ] Authorization checks implemented
- [ ] Passwords hashed with bcrypt
- [ ] CSRF protection enabled
- [ ] Logging configured (no secrets)
- [ ] Dependencies audited (`npm audit`)
- [ ] Error messages don't leak info
- [ ] Sensitive data not exposed in responses
- [ ] Database backups encrypted
- [ ] Monitoring/alerting configured

### Regular Maintenance

- [ ] Run `npm audit` regularly
- [ ] Review logs for anomalies
- [ ] Rotate encryption keys
- [ ] Update dependencies
- [ ] Review access logs
- [ ] Test security controls
- [ ] Penetration testing
- [ ] Security training for team
