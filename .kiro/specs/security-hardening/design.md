# Security Hardening - Design Document

## Overview

This document outlines the comprehensive security hardening implementation for BANKERCHANGER, a decentralized boxing prediction market. The design covers authentication/authorization hardening, data protection, API security, and compliance requirements.

The implementation follows defense-in-depth principles:
- **Authentication**: Multi-layer token lifecycle with refresh rotation, CSRF protection, and WebAuthn 2FA
- **Authorization**: Fine-grained RBAC with permission matrix and resource-level checks
- **Data Protection**: AES-256-GCM encryption at rest, TLS 1.3+ in transit, HSM-backed key storage
- **API Security**: Input validation, rate limiting, security headers, audit logging
- **Secrets Management**: Centralized vault with rotation and access logging

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                         API Gateway                              │
├─────────────────────────────────────────────────────────────────┤
│  Security Headers │ HTTPS/TLS 1.3 │ Rate Limit │ CSRF Token    │
└────────────────────────┬────────────────────────────────────────┘
                         │
        ┌────────────────┼────────────────┐
        │                │                │
    ┌───▼──────┐    ┌────▼─────┐    ┌────▼──────┐
    │  Auth    │    │  Request  │    │   Audit   │
    │ Middleware│   │ Validator │    │  Logger   │
    └───┬──────┘    └────┬─────┘    └────┬──────┘
        │                │               │
        │   JWT Token    │ Zod Schema    │ Event Stream
        │  (15m access)  │ Validation    │ to PostgreSQL
        │                │               │
    ┌───▼──────────────────────────────────────────┐
    │         RBAC Middleware + Permission Check    │
    │  (Role + Resource-level authorization)       │
    └───┬──────────────────────────────────────────┘
        │
    ┌───▼────────────────────────────────────┐
    │   Controller Layer (Route Handlers)     │
    └───┬────────────────────────────────────┘
        │
    ┌───▼──────────────────────────────────┐
    │   Service Layer (Business Logic)      │
    │  - Encryption/Decryption (AES-256)   │
    │  - Token Refresh Rotation             │
    │  - 2FA Validation (TOTP/WebAuthn)    │
    └───┬──────────────────────────────────┘
        │
    ┌───▼──────────────────────────────────┐
    │   Data Layer (PostgreSQL)             │
    │  - Encrypted user secrets             │
    │  - Audit event log (immutable)        │
    │  - Refresh token issuances            │
    └──────────────────────────────────────┘
```

## Components and Interfaces

### 1. CSRF Protection Module

**Purpose**: Prevent Cross-Site Request Forgery attacks on state-changing operations.

**Implementation Pattern**: Double-Submit Cookie + Token Validation

```typescript
interface CSRFTokenPayload {
  nonce: string;           // Cryptographically random nonce
  userId?: string;         // Optional: bind to user (bound-token pattern)
  timestamp: number;       // Issuance time (for rotation policy)
  httpOnly: boolean;       // Always true for HttpOnly cookies
}

interface CSRFConfig {
  tokenLifetime: number;   // 1 hour default
  headerName: string;      // X-CSRF-Token
  cookieName: string;      // csrf-token
  sameSite: 'Strict' | 'Lax' | 'None';  // Lax for most cases
  secure: boolean;         // true in production
  httpOnly: boolean;       // true (NOT accessed by JS)
}

// Middleware usage:
// app.use(generateCSRFToken());           // Generate on GET requests
// app.post('/action', validateCSRFToken(), handler); // Validate on POST/PUT/DELETE
```

**Key Design Decisions**:
- CSRF tokens are NOT user-specific to avoid session fixation attacks
- Tokens rotated on each request (strict policy) to minimize window of compromise
- Double-submit cookie pattern: token in both cookie (automatic by browser) and body/header (verified by attacker)
- SameSite=Lax protects most requests; SameSite=Strict for sensitive operations (admin)

### 2. JWT Token Lifecycle & Refresh Mechanism

**Purpose**: Minimize damage from token compromise while balancing user experience.

**Token Types**:
```typescript
interface JWTPayload {
  sub: string;             // Subject (user ID)
  type: 'access' | 'refresh' | 'temp' | 'password-reset' | 'ws_activity';
  iat: number;             // Issued at
  exp: number;             // Expires at
  sv?: number;             // Session version (invalidation)
  pv?: number;             // Password version (invalidation on password change)
  role?: string;           // 'user' | 'admin' | 'oracle' | 'moderator'
}

interface TokenConfig {
  // Access token: Short-lived, used for API requests
  ACCESS_TTL: '15m',       // 15 minutes
  ACCESS_SECRET: string;   // Separate from refresh secret
  
  // Refresh token: Longer-lived, used to obtain new access tokens
  REFRESH_TTL: '7d',       // 7 days
  REFRESH_SECRET: string;  // MUST be different from ACCESS_SECRET
  REFRESH_ROTATION: true,  // Issue new refresh token on use
  
  // Temporary token: For 2FA, email verification
  TEMP_TTL: '5m',
  TEMP_SECRET: string;
  
  // Password reset token: For account recovery
  RESET_TTL: '15m',
  RESET_SECRET: string;
  
  // WebSocket activity token: For real-time market feeds
  WS_ACTIVITY_TTL: '5m',
}
```

**Token Flow**:
```
1. User Login
   POST /auth/login { email, password, 2fa_code? }
   Response:
   {
     accessToken: "eyJhbGc...",      // 15m, in Authorization header
     refreshToken: "eyJhbGc...",     // 7d, in HttpOnly cookie
     accessTokenExpires: 900000,      // ms
     refreshTokenExpires: 604800000,  // ms
   }

2. Using Access Token
   GET /api/portfolio
   Header: Authorization: Bearer eyJhbGc...
   
   Middleware validates:
   - Signature (using ACCESS_SECRET)
   - Expiration
   - Type = 'access'
   - Session version (not revoked)
   - Password version (not stale)

3. Access Token Expires → Use Refresh Token
   POST /auth/refresh
   Cookie: refreshToken=eyJhbGc...
   
   Server validates:
   - Refresh token signature (using REFRESH_SECRET)
   - Expiration
   - Type = 'refresh'
   - Session version
   - Token not blacklisted (REVOKED)
   
   If valid:
   - Issue NEW refresh token (rotation)
   - Invalidate OLD refresh token
   - Return new access token
   - Cookie: refreshToken=eyJhbGM...new (replaces old)

4. Logout / Password Change
   DELETE /auth/sessions
   Server increments session_version in DB
   All existing access/refresh tokens invalidated
   (checked in middleware: sv != DB.session_version)
```

**Refresh Token Rotation Storage**:
```typescript
// PostgreSQL table: refresh_token_issuances
{
  id: serial,
  user_id: string,
  token_hash: string unique,  // SHA256(refresh_token)
  issued_at: timestamp,
  expires_at: timestamp,
  revoked_at: timestamp | null,  // Revoked on logout or password change
  created_at: timestamp
}

// On token use (POST /auth/refresh):
// 1. Find token by hash in DB
// 2. If revoked_at is set → Reject (possible token reuse attack)
// 3. Mark old token as revoked_at = NOW()
// 4. Issue new token with new hash
// 5. Insert new row
// 6. Delete refresh tokens older than 7 days (cron job)
```

**Key Design Decisions**:
- Access/Refresh secrets are separate (leaked access token can't forge refresh tokens)
- Refresh token rotation prevents stolen token reuse (only latest token is valid)
- Session versioning allows instant logout (no cache delay)
- Password versioning invalidates sessions on password change
- Refresh tokens stored as HttpOnly cookies (immune to XSS cookie theft)
- Token reuse detection: Old token used after new token issued = possible compromise alert

### 3. Role-Based Access Control (RBAC) & Permission Matrix

**Purpose**: Enforce principle of least privilege across all operations.

**Role Definitions**:
```typescript
enum Role {
  USER = 'user',           // Regular trading user
  ADMIN = 'admin',         // System administrator
  ORACLE = 'oracle',       // Oracle/reporter (blockchain)
  MODERATOR = 'moderator', // Content moderation
}

enum Permission {
  // Market operations
  CREATE_MARKET = 'market:create',
  UPDATE_MARKET = 'market:update',
  DELETE_MARKET = 'market:delete',
  VIEW_MARKET = 'market:view',
  RESOLVE_MARKET = 'market:resolve',
  
  // User operations
  VIEW_USER = 'user:view',
  UPDATE_USER = 'user:update',
  DELETE_USER = 'user:delete',
  MANAGE_ROLES = 'user:manage-roles',
  RESET_PASSWORD = 'user:reset-password',
  DISABLE_2FA = 'user:disable-2fa',
  
  // Admin operations
  VIEW_ADMIN_AUDIT = 'admin:view-audit',
  MANAGE_SECRETS = 'admin:manage-secrets',
  ROTATE_KEYS = 'admin:rotate-keys',
  VIEW_METRICS = 'admin:view-metrics',
  
  // Oracle operations
  SUBMIT_REPORT = 'oracle:submit-report',
  VIEW_REPORTS = 'oracle:view-reports',
  
  // Moderation
  FLAG_CONTENT = 'moderator:flag-content',
  SUSPEND_USER = 'moderator:suspend-user',
}

interface PermissionMatrix {
  [role: string]: Permission[];
}

const PERMISSION_MATRIX: PermissionMatrix = {
  [Role.USER]: [
    Permission.VIEW_MARKET,
    Permission.VIEW_USER,  // Own profile only
  ],
  [Role.ADMIN]: [
    // All permissions
    ...Object.values(Permission),
  ],
  [Role.ORACLE]: [
    Permission.VIEW_MARKET,
    Permission.SUBMIT_REPORT,
    Permission.VIEW_REPORTS,
  ],
  [Role.MODERATOR]: [
    Permission.VIEW_MARKET,
    Permission.FLAG_CONTENT,
    Permission.SUSPEND_USER,
    Permission.VIEW_ADMIN_AUDIT, // Limited to moderation events
  ],
};
```

**Resource-Level Permissions**:
```typescript
interface ResourcePermissionCheck {
  role: string;
  permission: Permission;
  resourceId?: string;       // Market ID, User ID, etc.
  resourceOwner?: string;    // If resource has owner
}

// Example: User can only view/update own profile
async function checkResourcePermission(
  userId: string,
  role: string,
  permission: Permission,
  resourceId: string,
  resourceType: 'user' | 'market' | 'order'
): Promise<boolean> {
  // 1. Check role-permission matrix
  if (!PERMISSION_MATRIX[role]?.includes(permission)) {
    return false;
  }
  
  // 2. Check resource-level permissions
  if (resourceType === 'user') {
    // User can only access own profile unless admin
    return userId === resourceId || role === Role.ADMIN;
  }
  
  if (resourceType === 'market') {
    // All authenticated users can view markets
    // Only admins can modify
    return permission !== Permission.UPDATE_MARKET || role === Role.ADMIN;
  }
  
  return false;
}
```

**RBAC Middleware**:
```typescript
// Usage in routes:
// app.post('/api/admin/users/:id/roles', 
//   requireAuth,
//   requirePermission(Permission.MANAGE_ROLES),
//   handler
// );

export function requirePermission(permission: Permission) {
  return async (req: Request, _res: Response, next: NextFunction) => {
    try {
      const userId = req.userId as string;
      const role = req.userRole as string;
      const resourceId = req.params.id;
      
      const allowed = await checkResourcePermission(
        userId,
        role,
        permission,
        resourceId,
        'user'
      );
      
      if (!allowed) {
        // Log permission denial for audit
        await auditLog.log({
          action: 'PERMISSION_DENIED',
          userId,
          resource: permission,
          resourceId,
          reason: 'Insufficient permissions',
        });
        
        return next(new AppError(403, 'Insufficient permissions'));
      }
      
      next();
    } catch (err) {
      next(err);
    }
  };
}
```

### 4. Encryption at Rest (AES-256-GCM)

**Purpose**: Protect sensitive data if database is compromised.

**Encrypted Fields**:
```typescript
interface EncryptedFields {
  // User secrets (highest priority)
  two_factor_secret: string,    // TOTP seed
  backup_codes: string[],       // 2FA recovery codes
  
  // Private keys (if stored)
  wallet_private_key?: string,
  
  // PII (optional, depends on compliance needs)
  // phone_number?: string,
}

interface EncryptionConfig {
  algorithm: 'aes-256-gcm',
  keyRotationInterval: '90d',   // Quarterly
  keyStorage: 'AWS KMS' | 'Vault' | 'Environment',
  masterKeyId: string,           // Key ID for rotation tracking
  additionalAuthData?: string,   // Optional AAD for authentication
}
```

**Encryption Implementation**:
```typescript
import crypto from 'crypto';

interface EncryptedData {
  ciphertext: string;      // Base64
  iv: string;              // Base64 (16 bytes random)
  authTag: string;         // Base64 (16 bytes GCM auth tag)
  algorithm: string;       // 'aes-256-gcm'
  keyId: string;           // For key rotation tracking
}

export function encrypt(
  plaintext: string,
  masterKey: Buffer,  // 32 bytes for AES-256
  keyId: string,
  aad?: string        // Additional authenticated data
): EncryptedData {
  const iv = crypto.randomBytes(16);
  const cipher = crypto.createCipheriv('aes-256-gcm', masterKey, iv);
  
  if (aad) cipher.setAAD(Buffer.from(aad));
  
  let ciphertext = cipher.update(plaintext, 'utf8', 'hex');
  ciphertext += cipher.final('hex');
  
  const authTag = cipher.getAuthTag();
  
  return {
    ciphertext: Buffer.from(ciphertext, 'hex').toString('base64'),
    iv: iv.toString('base64'),
    authTag: authTag.toString('base64'),
    algorithm: 'aes-256-gcm',
    keyId,
  };
}

export function decrypt(
  encrypted: EncryptedData,
  masterKey: Buffer,
  aad?: string
): string {
  const decipher = crypto.createDecipheriv(
    'aes-256-gcm',
    masterKey,
    Buffer.from(encrypted.iv, 'base64')
  );
  
  decipher.setAuthTag(Buffer.from(encrypted.authTag, 'base64'));
  if (aad) decipher.setAAD(Buffer.from(aad));
  
  let plaintext = decipher.update(
    Buffer.from(encrypted.ciphertext, 'base64')
  );
  plaintext = Buffer.concat([
    plaintext,
    decipher.final(),
  ]);
  
  return plaintext.toString('utf8');
}
```

**Key Storage Strategy**:
```typescript
// Option 1: AWS KMS (Recommended for production)
{
  provider: 'AWS KMS',
  masterKeyArn: 'arn:aws:kms:us-east-1:123456:key/12345',
  keyRotationDays: 90,
}

// Option 2: HashiCorp Vault
{
  provider: 'Vault',
  vaultAddr: 'https://vault.example.com',
  vaultRole: 'bankerchanger-app',
  vaultPath: 'secret/bankerchanger/master-key',
  keyRotationDays: 90,
}

// Option 3: Environment variable (Development only)
{
  provider: 'Environment',
  envVar: 'MASTER_ENCRYPTION_KEY',  // Base64-encoded 32 bytes
  keyRotationDays: 90,
}

// Key rotation process:
// 1. Generate new master key
// 2. Re-encrypt all encrypted fields with new key
// 3. Store old key in KMS/Vault with expiration
// 4. Update keyId in all encrypted records
// 5. Retire old key after 30 days retention
```

### 5. Encryption in Transit (HTTPS/TLS 1.3, WSS)

**Purpose**: Prevent man-in-the-middle attacks on network communication.

**Configuration**:
```typescript
interface TransitEncryptionConfig {
  // HTTP/HTTPS
  enforceHTTPS: true,                    // Redirect HTTP → HTTPS
  tlsMinVersion: 'TLSv1.3',              // No TLS 1.0, 1.1, 1.2
  tlsCipherSuites: [                     // Modern, secure ciphers
    'TLS_AES_256_GCM_SHA384',            // Recommended
    'TLS_CHACHA20_POLY1305_SHA256',      // Modern
    'TLS_AES_128_GCM_SHA256',            // Fallback
  ],
  certificatePinning: {
    enabled: true,                       // For critical endpoints
    pins: ['sha256/...', 'sha256/...'],  // Public key pins
  },
  
  // Headers
  hsts: {
    maxAge: 31536000,        // 1 year
    includeSubDomains: true,
    preload: true,
  },
  
  // WebSocket
  wsSecure: true,            // WSS only, never WS
}

// Express middleware:
app.use((req, res, next) => {
  // Redirect HTTP to HTTPS
  if (req.header('x-forwarded-proto') !== 'https' && 
      process.env.NODE_ENV === 'production') {
    return res.redirect(301, `https://${req.header('host')}${req.url}`);
  }
  
  // Set HSTS header
  res.set('Strict-Transport-Security', 
    'max-age=31536000; includeSubDomains; preload');
  
  next();
});
```

### 6. Secrets Management

**Purpose**: Centralize secret storage, rotation, and access auditing.

**Secrets to Manage**:
```typescript
interface ManagedSecrets {
  // Authentication
  JWT_ACCESS_SECRET: string,
  JWT_REFRESH_SECRET: string,
  JWT_TEMP_SECRET: string,
  JWT_RESET_SECRET: string,
  
  // Encryption
  MASTER_ENCRYPTION_KEY: string,         // Base64-encoded 32 bytes
  
  // External services
  SENDGRID_API_KEY: string,
  AWS_KMS_KEY_ID: string,
  DATABASE_URL: string,
  REDIS_URL: string,
  
  // OAuth (if applicable)
  GOOGLE_CLIENT_SECRET: string,
  GITHUB_CLIENT_SECRET: string,
  
  // Blockchain
  STELLAR_SIGNING_KEY: string,           // For indexer
  ORACLE_SIGNING_KEY: string,
}

interface SecretsConfig {
  // Option 1: AWS Secrets Manager
  provider: 'AWS',
  region: 'us-east-1',
  rotationInterval: '90d',
  autoRotation: true,
  
  // Option 2: HashiCorp Vault
  provider: 'Vault',
  vaultAddr: 'https://vault.example.com:8200',
  vaultNamespace: 'bankerchanger',
  rotationInterval: '90d',
  
  // Option 3: Environment variables (Dev/Staging)
  provider: 'Environment',
  rotationInterval: 'manual',
}
```

**Secret Rotation Procedure**:
```typescript
// Automated quarterly rotation
async function rotateSecrets() {
  for (const secretName of Object.keys(ManagedSecrets)) {
    // 1. Generate new secret
    const newSecret = generateRandomSecret();
    
    // 2. Store new secret in vault (versioned)
    await vault.createSecretVersion(secretName, {
      value: newSecret,
      rotatedAt: new Date(),
      previousVersions: [1, 2, 3],  // Keep last 3 versions
    });
    
    // 3. Trigger dependent rotations (JWTs, encryption keys)
    if (secretName.includes('JWT')) {
      // Update all active sessions with new JWT secret
      // Old tokens still valid until expiration
      await notifyApplicationReload();
    }
    
    if (secretName === 'MASTER_ENCRYPTION_KEY') {
      // Re-encrypt all encrypted fields
      await reEncryptAllFields();
    }
    
    // 4. Log rotation event
    await auditLog.log({
      action: 'SECRET_ROTATED',
      secretName,
      rotatedBy: 'SYSTEM',
      previousVersion: 2,
      newVersion: 3,
    });
  }
}
```

**Access Control**:
```typescript
// Only specific services can access secrets
// Enforce with IAM policies or Vault roles

// AWS IAM policy:
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "secretsmanager:GetSecretValue",
      "Resource": "arn:aws:secretsmanager:us-east-1:*:secret:bankerchanger/*",
      "Condition": {
        "StringEquals": {
          "aws:iam:SourceVpc": "vpc-12345"
        }
      }
    }
  ]
}

// Vault policy:
{
  "path": "secret/bankerchanger/*" {
    "capabilities": ["read"]
  },
  "path": "secret/bankerchanger/metadata" {
    "capabilities": ["list"]
  }
}
```

### 7. Input Validation & Output Encoding

**Purpose**: Prevent injection attacks (SQL injection, XSS, command injection).

**Validation Framework**: Zod (already in stack)

```typescript
import { z } from 'zod';

// User input schemas
const CreateUserSchema = z.object({
  email: z.string()
    .email('Invalid email format')
    .max(254, 'Email too long'),
  password: z.string()
    .min(12, 'Password must be at least 12 characters')
    .regex(/[A-Z]/, 'Password must contain uppercase letter')
    .regex(/[0-9]/, 'Password must contain digit')
    .regex(/[!@#$%^&*]/, 'Password must contain special character'),
  username: z.string()
    .min(3, 'Username must be at least 3 characters')
    .max(50, 'Username too long')
    .regex(/^[a-zA-Z0-9_-]+$/, 'Username can only contain alphanumeric, underscore, hyphen'),
});

const BetSchema = z.object({
  marketId: z.string().uuid('Invalid market ID'),
  amount: z.number()
    .positive('Amount must be positive')
    .max(1000000, 'Amount exceeds maximum'),
  side: z.enum(['A', 'B', 'DRAW']),
});

// Validation middleware
export function validateRequest(schema: z.ZodSchema) {
  return (req: Request, res: Response, next: NextFunction) => {
    try {
      const validated = schema.parse(req.body);
      req.validatedData = validated;
      next();
    } catch (err) {
      if (err instanceof z.ZodError) {
        // Never leak validation details that could help attackers
        // Log full error for debugging
        logger.warn({ err }, 'Validation failed');
        
        // Return generic error to client
        return next(new AppError(400, 'Invalid input'));
      }
      next(err);
    }
  };
}

// Output encoding
export function encodeHTMLOutput(text: string): string {
  return text
    .replace(/&/g, '&amp;')
    .replace(/</g, '&lt;')
    .replace(/>/g, '&gt;')
    .replace(/"/g, '&quot;')
    .replace(/'/g, '&#x27;');
}

// JSON response validation
export function validateResponse<T>(data: unknown, schema: z.ZodSchema): T {
  try {
    return schema.parse(data);
  } catch (err) {
    logger.error({ err }, 'Response validation failed - data integrity issue');
    throw new Error('Internal server error');
  }
}
```

**File Upload Validation**:
```typescript
interface FileUploadConfig {
  maxSize: 5 * 1024 * 1024,          // 5MB
  allowedMimeTypes: [
    'image/jpeg',
    'image/png',
    'application/pdf',
  ],
  scanForMalware: true,              // Use ClamAV or similar
}

export async function validateFileUpload(
  file: Express.Multer.File
): Promise<boolean> {
  // 1. Check size
  if (file.size > 5 * 1024 * 1024) {
    throw new AppError(413, 'File too large');
  }
  
  // 2. Check MIME type
  const allowedTypes = ['image/jpeg', 'image/png', 'application/pdf'];
  if (!allowedTypes.includes(file.mimetype)) {
    throw new AppError(400, 'Invalid file type');
  }
  
  // 3. Verify file signature (magic bytes)
  const buffer = await fs.promises.readFile(file.path);
  const signature = buffer.slice(0, 4).toString('hex');
  const validSignatures = {
    'ffd8ffe0': 'image/jpeg',
    '89504e47': 'image/png',
    '25504446': 'application/pdf',
  };
  
  if (!validSignatures[signature]) {
    throw new AppError(400, 'Invalid file content');
  }
  
  // 4. Scan for malware (if enabled)
  if (process.env.ENABLE_MALWARE_SCAN === 'true') {
    const isSafe = await scanForMalware(file.path);
    if (!isSafe) {
      throw new AppError(400, 'File contains malicious content');
    }
  }
  
  return true;
}
```

### 8. Rate Limiting & Brute Force Protection

**Purpose**: Prevent brute force attacks, DoS, and resource exhaustion.

**Configuration**:
```typescript
interface RateLimitConfig {
  // Login attempts
  login: {
    maxAttempts: 5,
    windowMs: 15 * 60 * 1000,  // 15 minutes
    lockoutDurationMs: 15 * 60 * 1000,  // 15 minute lockout
    captchaThreshold: 3,  // Require CAPTCHA after 3 failures
  },
  
  // API endpoints (general)
  api: {
    maxRequests: 100,
    windowMs: 60 * 1000,  // 60 seconds
    keyBy: 'ip',  // By IP or user ID
  },
  
  // Sensitive endpoints
  admin: {
    maxRequests: 20,
    windowMs: 60 * 1000,
    keyBy: 'ip',
  },
  
  // Password reset
  passwordReset: {
    maxAttempts: 3,
    windowMs: 60 * 60 * 1000,  // 1 hour
  },
  
  // 2FA attempts
  twoFa: {
    maxAttempts: 5,
    windowMs: 15 * 60 * 1000,  // 15 minutes
  },
}

// Redis-backed rate limiter (already implemented in code)
export async function checkRateLimit(
  key: string,
  max: number,
  windowSec: number
): Promise<{ allowed: boolean; remaining: number; resetTime: number }> {
  const count = await redis.incr(key);
  
  if (count === 1) {
    await redis.expire(key, windowSec);
  }
  
  const ttl = await redis.ttl(key);
  
  return {
    allowed: count <= max,
    remaining: Math.max(0, max - count),
    resetTime: ttl,
  };
}

// Distributed attack detection
export async function detectDistributedAttack(
  resource: string,  // e.g., 'login'
  maxUniqueIPs: number = 10
): Promise<boolean> {
  // If 10+ different IPs attempt same resource in short time = distributed attack
  const key = `attack_detection:${resource}`;
  const ips = await redis.smembers(key);
  
  return ips.length > maxUniqueIPs;
}
```

### 9. Audit Logging

**Purpose**: Track all sensitive operations for compliance and forensics.

**Audit Log Schema**:
```typescript
// PostgreSQL table: security_audit_logs
interface AuditLog {
  id: serial,
  
  // Event metadata
  action: string,                    // LOGIN, PERMISSION_DENIED, SECRET_ROTATED, etc.
  timestamp: timestamp,              // When event occurred
  
  // Actor
  actor_id: string,                  // User ID, service name
  actor_type: 'user' | 'service',
  actor_role: string,                // User's role at time of action
  
  // Resource
  resource_type: string,             // 'user', 'market', 'secret', 'key', etc.
  resource_id: string,               // ID of resource affected
  
  // Action details
  action_status: 'success' | 'failure',
  action_reason?: string,            // Why action was taken/denied
  details: jsonb,                    // Additional context
  
  // Security context
  ip_address: inet,                  // Source IP
  user_agent: text,                  // Browser/client info
  session_id: uuid,                  // Session for correlation
  
  // Compliance
  severity: 'info' | 'warning' | 'critical',
}

// Indexes for efficient queries
CREATE INDEX audit_logs_timestamp_idx ON security_audit_logs(timestamp DESC);
CREATE INDEX audit_logs_actor_id_idx ON security_audit_logs(actor_id);
CREATE INDEX audit_logs_action_idx ON security_audit_logs(action);
CREATE INDEX audit_logs_resource_idx ON security_audit_logs(resource_type, resource_id);
```

**Audit Event Types**:
```typescript
enum AuditAction {
  // Authentication
  LOGIN = 'LOGIN',
  LOGOUT = 'LOGOUT',
  LOGIN_FAILED = 'LOGIN_FAILED',
  PASSWORD_CHANGED = 'PASSWORD_CHANGED',
  PASSWORD_RESET = 'PASSWORD_RESET',
  
  // 2FA
  TOTP_ENABLED = 'TOTP_ENABLED',
  TOTP_DISABLED = 'TOTP_DISABLED',
  WEBAUTHN_ADDED = 'WEBAUTHN_ADDED',
  WEBAUTHN_REMOVED = 'WEBAUTHN_REMOVED',
  '2FA_FAILURE' = '2FA_FAILURE',
  
  // Authorization
  PERMISSION_DENIED = 'PERMISSION_DENIED',
  ROLE_CHANGED = 'ROLE_CHANGED',
  
  // Secrets & Keys
  SECRET_ROTATED = 'SECRET_ROTATED',
  ENCRYPTION_KEY_ROTATED = 'ENCRYPTION_KEY_ROTATED',
  SECRET_ACCESSED = 'SECRET_ACCESSED',
  
  // Admin actions
  USER_SUSPENDED = 'USER_SUSPENDED',
  USER_DELETED = 'USER_DELETED',
  MARKET_RESOLVED = 'MARKET_RESOLVED',
  
  // Security incidents
  BRUTE_FORCE_ATTEMPT = 'BRUTE_FORCE_ATTEMPT',
  CSRF_TOKEN_MISMATCH = 'CSRF_TOKEN_MISMATCH',
  SUSPICIOUS_ACTIVITY = 'SUSPICIOUS_ACTIVITY',
}

// Logging helper
export async function auditLog(event: {
  action: AuditAction;
  actor: { id: string; type: 'user' | 'service'; role?: string };
  resource?: { type: string; id: string };
  status: 'success' | 'failure';
  reason?: string;
  details?: Record<string, any>;
  ip?: string;
  userAgent?: string;
}): Promise<void> {
  try {
    await db.insert(security_audit_logs).values({
      action: event.action,
      actor_id: event.actor.id,
      actor_type: event.actor.type,
      actor_role: event.actor.role,
      resource_type: event.resource?.type,
      resource_id: event.resource?.id,
      action_status: event.status,
      action_reason: event.reason,
      details: event.details,
      ip_address: event.ip,
      user_agent: event.userAgent,
      timestamp: new Date(),
      severity: determineSeverity(event.action, event.status),
    });
  } catch (err) {
    logger.error({ err, event }, 'Failed to write audit log');
    // Continue operation even if audit log fails (don't block user)
  }
}

// Querying audit logs for investigations
export async function getAuditTrail(
  resourceType: string,
  resourceId: string,
  limit: number = 100
): Promise<AuditLog[]> {
  return db.query.security_audit_logs
    .findMany({
      where: and(
        eq(schema.security_audit_logs.resource_type, resourceType),
        eq(schema.security_audit_logs.resource_id, resourceId)
      ),
      orderBy: desc(schema.security_audit_logs.timestamp),
      limit,
    });
}
```

## Correctness Properties

*A property is a characteristic or behavior that should hold true across all valid executions of a system—essentially, a formal statement about what the system should do. Properties serve as the bridge between human-readable specifications and machine-verifiable correctness guarantees.*

Given the infrastructure-heavy nature of this security hardening feature (HTTPS/TLS configuration, secrets vault integration, cloud IAM policies, and database audit logging), comprehensive property-based testing has limited applicability. Instead, the design relies on **integration tests with representative examples** and **configuration verification tests**.

However, where pure business logic exists, property-based tests ARE appropriate:

### Property 1: CSRF Token Double-Submit Consistency

*For any* state-changing HTTP request with valid CSRF token in both cookie and body/header, the request SHALL be accepted.

**Validates: Requirements 1 (CSRF token validation middleware)**

### Property 2: Token Rotation Idempotence

*For any* valid refresh token, calling the refresh endpoint twice SHALL issue two distinct new tokens, and only the most recent token SHALL be valid for future requests.

**Validates: Requirements 2 (Refresh token rotation)**

### Property 3: Permission Matrix Enforcement

*For any* user with role R attempting action A on resource X, the action SHALL succeed if and only if permission matrix allows (R → A).

**Validates: Requirements 3 (RBAC permission checks)**

### Property 4: Encryption Round-Trip Preservation

*For any* plaintext secret, encrypting then decrypting SHALL return the original plaintext unchanged.

**Validates: Requirements 4 (AES-256-GCM encryption)**

### Property 5: Audit Log Immutability

*For any* audit log entry written to the database, reading that entry back SHALL return identical data (no fields modified, no timestamps altered).

**Validates: Requirements 6 (Audit logging implementation)**

## Error Handling

**Security-Specific Error Handling**:
```typescript
// NEVER expose internal details in error responses
class SecurityError extends AppError {
  constructor(
    statusCode: number,
    publicMessage: string,  // Safe to show user
    internalDetails?: Record<string, any>
  ) {
    super(statusCode, publicMessage);
    
    // Log internal details for debugging
    logger.error({ internalDetails }, publicMessage);
  }
}

// Examples:
// ❌ Bad: "Invalid JWT signature: expected RS256, got none"
// ✅ Good: "Invalid or expired token"

// ❌ Bad: "User login_attempt rate limit exceeded: 8/5 attempts"
// ✅ Good: "Too many login attempts. Please try again in 15 minutes."

// ❌ Bad: "SQL error: column 'password' does not exist"
// ✅ Good: "An error occurred. Please try again."

// Timing attack prevention: Use constant-time comparison
import { timingSafeEqual } from 'crypto';

function verifyToken(providedToken: string, storedHash: string): boolean {
  try {
    return timingSafeEqual(
      Buffer.from(providedToken),
      Buffer.from(storedHash)
    );
  } catch {
    // Lengths don't match; fail safely without timing leak
    return false;
  }
}
```

## Testing Strategy

### Property-Based Tests (When Applicable)

Property-based tests validate pure business logic with high iteration counts (100+):

1. **CSRF Token Validation** (if token generation/validation is pure function)
   - Generate random CSRF tokens
   - Verify double-submit property
   - Property test: `roundTrip(generate, validate) == true`

2. **Token Rotation** (if refresh logic is testable in isolation)
   - Generate random valid/invalid refresh tokens
   - Verify rotation replaces old token with new
   - Property test: `newToken != oldToken && oldToken.invalid() == true`

3. **Permission Matrix** (permission lookup is pure logic)
   - Generate random user/role/action combinations
   - Verify against PERMISSION_MATRIX
   - Property test: `canPerform(role, action) == PERMISSION_MATRIX[role].includes(action)`

4. **Encryption Round-Trip** (encrypt/decrypt is pure)
   - Generate random plaintext strings
   - Verify `decrypt(encrypt(x)) == x`
   - Property test with edge cases: empty string, UTF-8, special chars, large data

5. **Audit Log Queries** (immutability verification)
   - Insert random audit events
   - Read them back immediately
   - Verify no fields were modified
   - Property test: `readBack(inserted) == inserted`

### Unit Tests

Specific examples, edge cases, error conditions:

1. **CSRF Token Expiration**
   ```typescript
   test('CSRF token older than 1 hour should be rejected', async () => {
     const oldToken = generateCSRFToken();
     // Manually set issuance time to 61 minutes ago
     await sleep(61 * 60 * 1000);
     expect(() => validateCSRFToken(oldToken)).toThrow('Token expired');
   });
   ```

2. **JWT Session Invalidation**
   ```typescript
   test('After logout, old access token should be rejected', async () => {
     const token = await login(user);
     await logout(user);
     expect(() => validateToken(token)).toThrow('Session invalidated');
   });
   ```

3. **2FA Bypass Prevention**
   ```typescript
   test('Cannot login with valid password but invalid 2FA code', async () => {
     const user = await createUserWith2FA();
     expect(() => login(user.email, user.password, 'invalid-code')).toThrow();
   });
   ```

4. **Rate Limit Enforcement**
   ```typescript
   test('6th login attempt within 15 minutes should be blocked', async () => {
     for (let i = 0; i < 5; i++) {
       await loginFail(email);
     }
     // 6th attempt should be rate limited
     expect(() => loginFail(email)).toThrow('Too many attempts');
   });
   ```

5. **Permission Denials Logged**
   ```typescript
   test('Permission denial should create audit log entry', async () => {
     const user = await createUser({ role: 'user' });
     await expect(deleteUser(user.id)).rejects.toThrow('Insufficient permissions');
     const audit = await getAuditLog({ action: 'PERMISSION_DENIED' });
     expect(audit).toHaveLength(1);
     expect(audit[0].resource_id).toBe(user.id);
   });
   ```

### Integration Tests

Infrastructure wiring, external services:

1. **HTTPS Enforcement**
   - Send HTTP request
   - Verify redirect to HTTPS
   - Verify TLS version 1.3 negotiated

2. **Secrets Vault Integration**
   - Start app with vault unavailable
   - Verify graceful degradation or error handling
   - Verify retry logic works after vault recovers

3. **Audit Log Persistence**
   - Perform security action
   - Query database directly
   - Verify audit entry exists with correct details

4. **2FA WebAuthn Enrollment**
   - Call WebAuthn endpoint
   - Verify challenge is cryptographically random
   - Verify challenge can only be used once

### Security Testing (Manual/Penetration Testing)

Covered by separate security testing plan:
- OWASP Top 10 compliance
- SQL injection attempts
- XSS payload testing
- CSRF attack simulation
- Brute force attack detection
- Penetration testing

## API Security Headers Configuration

All responses should include these security headers:

```typescript
app.use((req, res, next) => {
  // Prevent clickjacking
  res.set('X-Frame-Options', 'DENY');
  
  // Prevent MIME type sniffing
  res.set('X-Content-Type-Options', 'nosniff');
  
  // Content Security Policy (CSP)
  res.set('Content-Security-Policy', 
    "default-src 'self'; " +
    "script-src 'self' cdn.example.com; " +
    "style-src 'self' 'unsafe-inline'; " +
    "img-src 'self' data: https:; " +
    "font-src 'self' https://fonts.googleapis.com; " +
    "connect-src 'self' https://api.example.com; " +
    "frame-ancestors 'none'; " +
    "upgrade-insecure-requests"
  );
  
  // Referrer Policy
  res.set('Referrer-Policy', 'strict-origin-when-cross-origin');
  
  // Feature-Policy (Permissions-Policy in modern browsers)
  res.set('Permissions-Policy', 
    'geolocation=(), ' +
    'microphone=(), ' +
    'camera=(), ' +
    'usb=(), ' +
    'magnetometer=(), ' +
    'gyroscope=(), ' +
    'accelerometer=()'
  );
  
  // HSTS (already set above, but repeated for clarity)
  res.set('Strict-Transport-Security', 
    'max-age=31536000; includeSubDomains; preload'
  );
  
  next();
});
```

## Implementation Notes

1. **Backward Compatibility**: Existing tokens should remain valid. New token validation happens gradually.
2. **Key Rotation**: Plan quarterly key rotation. Implement versioning to support multiple key versions during rotation.
3. **Performance**: Encryption/decryption should add < 1% overhead. Use connection pooling for database queries.
4. **Monitoring**: Set up alerts for suspicious patterns (e.g., many failed login attempts, unusual access to secrets).
5. **Documentation**: Create runbooks for:
   - Emergency secret rotation
   - Key recovery procedures
   - Audit log investigation
   - 2FA recovery for locked-out users
