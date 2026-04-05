# Authentication Workflows

This document covers complete authentication implementation patterns.

## Registration Flow

### Registration DTO

```typescript
// src/auth/dto/register.dto.ts
import {
  IsEmail,
  IsString,
  MinLength,
  Matches,
  IsInt,
  Min,
  Max,
} from 'class-validator';

export class RegisterDto {
  @IsEmail()
  email: string;

  @IsString()
  @MinLength(2)
  name: string;

  @IsString()
  @MinLength(12)
  @Matches(/[A-Z]/, { message: 'Must contain uppercase' })
  @Matches(/[a-z]/, { message: 'Must contain lowercase' })
  @Matches(/[0-9]/, { message: 'Must contain number' })
  @Matches(/[!@#$%^&*]/, { message: 'Must contain special char' })
  password: string;

  @IsInt()
  @Min(13)
  @Max(150)
  age: number;
}
```

### Registration Service

```typescript
// src/auth/auth.service.ts
@Injectable()
export class AuthService {
  constructor(
    private readonly userRepository: UserRepository,
    private readonly passwordService: PasswordService,
    private readonly tokenService: AuthTokenService,
    private readonly logger: Logger,
  ) {}

  async register(dto: RegisterDto): Promise<{ accessToken: string; refreshToken: string }> {
    // Check email exists
    const existing = await this.userRepository.findByEmail(dto.email);
    if (existing) {
      this.logger.warn(`Registration attempt with existing email: ${dto.email}`);
      throw new EmailAlreadyInUseError(dto.email);
    }

    // Validate password strength
    const strength = this.passwordService.validateStrength(dto.password);
    if (!strength.valid) {
      throw new InvalidPasswordError(strength.errors.join(', '));
    }

    // Hash password
    const passwordHash = await this.passwordService.hash(dto.password);

    // Create user
    const user = new User();
    user.email = dto.email;
    user.name = dto.name;
    user.age = dto.age;
    user.passwordHash = passwordHash;
    user.roles = ['user']; // Default role

    const saved = await this.userRepository.save(user);

    this.logger.log(`User registered: ${saved.id}`);

    // Generate tokens
    const accessToken = this.tokenService.generateAccessToken(
      saved.id,
      saved.email,
      saved.roles,
    );
    const refreshToken = this.tokenService.generateRefreshToken(saved.id);

    return { accessToken, refreshToken };
  }
}
```

### Registration Controller

```typescript
// src/auth/auth.controller.ts
@Controller('auth')
export class AuthController {
  constructor(private readonly authService: AuthService) {}

  @Post('register')
  async register(@Body() dto: RegisterDto) {
    return this.authService.register(dto);
  }
}
```

## Login Flow

### Login DTO

```typescript
// src/auth/dto/login.dto.ts
import { IsEmail, IsString } from 'class-validator';

export class LoginDto {
  @IsEmail()
  email: string;

  @IsString()
  password: string;
}
```

### Login Service

```typescript
@Injectable()
export class AuthService {
  private loginAttempts = new Map<string, { count: number; lastAttempt: Date }>();

  async login(dto: LoginDto): Promise<{ accessToken: string; refreshToken: string }> {
    // Check login attempts
    const attempts = this.loginAttempts.get(dto.email);
    if (attempts && attempts.count > 5) {
      const timeSinceLastAttempt = Date.now() - attempts.lastAttempt.getTime();
      if (timeSinceLastAttempt < 15 * 60 * 1000) { // 15 minutes
        this.logger.warn(`Account locked: ${dto.email}`);
        throw new TooManyLoginAttemptsError(15);
      } else {
        // Reset after 15 minutes
        this.loginAttempts.delete(dto.email);
      }
    }

    // Find user
    const user = await this.userRepository.findByEmail(dto.email);

    // Verify credentials (don't reveal if user exists)
    if (!user || !(await this.passwordService.verify(dto.password, user.passwordHash))) {
      // Log failed attempt
      this.loginAttempts.set(dto.email, {
        count: (attempts?.count || 0) + 1,
        lastAttempt: new Date(),
      });

      this.logger.warn(`Failed login: ${dto.email}`);
      throw new CredentialsInvalidError(); // Generic message
    }

    // Reset attempts on success
    this.loginAttempts.delete(dto.email);

    this.logger.log(`User logged in: ${user.id}`);

    // Generate tokens
    const accessToken = this.tokenService.generateAccessToken(
      user.id,
      user.email,
      user.roles,
    );
    const refreshToken = this.tokenService.generateRefreshToken(user.id);

    // Store refresh token (optional - for logout)
    await this.refreshTokenRepository.create({
      userId: user.id,
      token: refreshToken,
      expiresAt: new Date(Date.now() + 7 * 24 * 60 * 60 * 1000), // 7 days
    });

    return { accessToken, refreshToken };
  }
}
```

### Login Controller

```typescript
@Post('login')
@UseGuards(RateLimitGuard) // Additional rate limiting
async login(@Body() dto: LoginDto) {
  return this.authService.login(dto);
}
```

## Token Refresh

### Refresh DTO

```typescript
// src/auth/dto/refresh.dto.ts
import { IsString } from 'class-validator';

export class RefreshDto {
  @IsString()
  refreshToken: string;
}
```

### Refresh Service

```typescript
async refreshTokens(refreshToken: string): Promise<{ accessToken: string }> {
  try {
    // Validate refresh token
    const payload = this.tokenService.validateRefreshToken(refreshToken);

    // Verify token exists in database
    const stored = await this.refreshTokenRepository.findOne({
      userId: payload.sub,
      token: refreshToken,
    });

    if (!stored || new Date() > stored.expiresAt) {
      throw new InvalidRefreshTokenError();
    }

    // Get user
    const user = await this.userRepository.findById(payload.sub);
    if (!user) {
      throw new UserNotFoundError(payload.sub);
    }

    // Generate new access token
    const accessToken = this.tokenService.generateAccessToken(
      user.id,
      user.email,
      user.roles,
    );

    this.logger.debug(`Token refreshed for user: ${user.id}`);

    return { accessToken };
  } catch (error) {
    this.logger.warn(`Token refresh failed: ${error.message}`);
    throw new InvalidRefreshTokenError();
  }
}
```

### Refresh Controller

```typescript
@Post('refresh')
async refresh(@Body() dto: RefreshDto) {
  return this.authService.refreshTokens(dto.refreshToken);
}
```

## Logout Flow

### Logout Service

```typescript
async logout(userId: string): Promise<void> {
  // Invalidate all refresh tokens
  await this.refreshTokenRepository.deleteMany({ userId });

  // Optionally: Add access token to blacklist
  // This prevents token reuse if compromised

  this.logger.log(`User logged out: ${userId}`);
}
```

### Logout Controller

```typescript
@Post('logout')
@UseGuards(JwtAuthGuard)
async logout(@Request() req) {
  await this.authService.logout(req.user.id);
  return { message: 'Logged out successfully' };
}
```

## Password Reset Flow

### Reset Request DTO

```typescript
// src/auth/dto/reset-password-request.dto.ts
import { IsEmail } from 'class-validator';

export class ResetPasswordRequestDto {
  @IsEmail()
  email: string;
}
```

### Reset Request Service

```typescript
async requestPasswordReset(email: string): Promise<void> {
  const user = await this.userRepository.findByEmail(email);

  // Don't reveal if email exists (security)
  if (!user) {
    this.logger.warn(`Password reset requested for non-existent email: ${email}`);
    return;
  }

  // Generate reset token (short-lived)
  const resetToken = this.generateSecureToken();
  const resetTokenHash = await bcrypt.hash(resetToken, 10);
  const expiresAt = new Date(Date.now() + 60 * 60 * 1000); // 1 hour

  // Store reset token
  await this.passwordResetRepository.create({
    userId: user.id,
    tokenHash: resetTokenHash,
    expiresAt,
  });

  // Send email (implement your email service)
  const resetUrl = `${process.env.FRONTEND_URL}/reset-password?token=${resetToken}&email=${email}`;

  this.logger.log(`Password reset requested for: ${email}`);
  // await this.emailService.sendPasswordReset(email, resetUrl);
}
```

### Reset Confirmation DTO

```typescript
// src/auth/dto/reset-password.dto.ts
import { IsString, MinLength, Matches } from 'class-validator';

export class ResetPasswordDto {
  @IsString()
  token: string;

  @IsString()
  @MinLength(12)
  @Matches(/[A-Z]/)
  @Matches(/[a-z]/)
  @Matches(/[0-9]/)
  @Matches(/[!@#$%^&*]/)
  newPassword: string;
}
```

### Reset Service

```typescript
async resetPassword(
  email: string,
  resetToken: string,
  newPassword: string,
): Promise<void> {
  // Find user
  const user = await this.userRepository.findByEmail(email);
  if (!user) {
    throw new UserNotFoundError(email);
  }

  // Find reset token
  const reset = await this.passwordResetRepository.findOne({
    userId: user.id,
  });

  if (!reset || new Date() > reset.expiresAt) {
    throw new InvalidTokenError('Reset token expired');
  }

  // Verify token
  const isValid = await bcrypt.compare(resetToken, reset.tokenHash);
  if (!isValid) {
    throw new InvalidTokenError('Invalid reset token');
  }

  // Validate new password
  const strength = this.passwordService.validateStrength(newPassword);
  if (!strength.valid) {
    throw new InvalidPasswordError(strength.errors.join(', '));
  }

  // Hash new password
  const passwordHash = await this.passwordService.hash(newPassword);

  // Update password
  await this.userRepository.update(user.id, { passwordHash });

  // Invalidate reset token
  await this.passwordResetRepository.delete({ userId: user.id });

  // Invalidate all refresh tokens (force re-login)
  await this.refreshTokenRepository.deleteMany({ userId: user.id });

  this.logger.log(`Password reset completed for: ${email}`);
}
```

## Session Security

### Secure Session Options

```typescript
// src/main.ts
import session from 'express-session';
import RedisStore from 'connect-redis';
import { createClient } from 'redis';

const redisClient = createClient();

app.use(
  session({
    store: new RedisStore({ client: redisClient }),
    secret: process.env.SESSION_SECRET,
    resave: false,
    saveUninitialized: false,
    cookie: {
      secure: process.env.NODE_ENV === 'production', // HTTPS only
      httpOnly: true,                                  // No JavaScript access
      sameSite: 'strict',                             // CSRF protection
      maxAge: 24 * 60 * 60 * 1000,                   // 24 hours
    },
  })
);
```

## Multi-Factor Authentication (MFA)

### MFA Setup

```typescript
// src/auth/mfa.service.ts
import speakeasy from 'speakeasy';
import QRCode from 'qrcode';

@Injectable()
export class MfaService {
  async generateSecret(email: string): Promise<{ secret: string; qrCode: string }> {
    const secret = speakeasy.generateSecret({
      name: `YourApp (${email})`,
      length: 32,
    });

    const qrCode = await QRCode.toDataURL(secret.otpauth_url);

    return {
      secret: secret.base32,
      qrCode,
    };
  }

  verifyToken(secret: string, token: string): boolean {
    return speakeasy.totp.verify({
      secret,
      encoding: 'base32',
      token,
      window: 2, // Allow tokens from ±2 time windows
    });
  }
}
```

### MFA Enable

```typescript
@Post('mfa/enable')
@UseGuards(JwtAuthGuard)
async enableMfa(@Request() req) {
  const { secret, qrCode } = await this.mfaService.generateSecret(req.user.email);

  // Store secret temporarily (pending verification)
  await this.mfaRepository.create({
    userId: req.user.id,
    secret,
    verified: false,
  });

  return { qrCode, secret }; // Show QR code to user
}

@Post('mfa/verify')
@UseGuards(JwtAuthGuard)
async verifyMfa(@Request() req, @Body('token') token: string) {
  const mfa = await this.mfaRepository.findOne({
    userId: req.user.id,
    verified: false,
  });

  if (!mfa) {
    throw new BadRequestException('No pending MFA setup');
  }

  const isValid = this.mfaService.verifyToken(mfa.secret, token);
  if (!isValid) {
    throw new BadRequestException('Invalid MFA token');
  }

  // Mark as verified
  await this.mfaRepository.update({ userId: req.user.id }, { verified: true });

  this.logger.log(`MFA enabled for user: ${req.user.id}`);

  return { message: 'MFA enabled' };
}
```

### MFA Login

```typescript
async loginWithMfa(dto: LoginDto, mfaToken: string): Promise<Tokens> {
  // Standard login
  const user = await this.userRepository.findByEmail(dto.email);

  if (!user || !(await this.passwordService.verify(dto.password, user.passwordHash))) {
    throw new CredentialsInvalidError();
  }

  // Verify MFA if enabled
  if (user.mfaEnabled) {
    const mfa = await this.mfaRepository.findOne({ userId: user.id, verified: true });

    if (!mfa || !this.mfaService.verifyToken(mfa.secret, mfaToken)) {
      throw new BadRequestException('Invalid MFA token');
    }
  }

  // Generate tokens
  return this.generateTokens(user);
}
```
