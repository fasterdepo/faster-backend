# Implementation Plan: Security Hardening

## Overview

This plan breaks down the Security Hardening design into incremental implementation steps for BANKERCHANGER. The implementation follows a bottom-up approach:

1. **Foundation**: Set up encryption/key management and database schema
2. **Authentication**: Implement CSRF protection, JWT token lifecycle, and 2FA hardening
3. **Authorization**: Build RBAC permission matrix and middleware
4. **API Security**: Add input validation, rate limiting, and security headers
5. **Audit & Compliance**: Implement audit logging, secrets management, and monitoring

Each task builds on previous tasks. All code is written in **TypeScript** (Node.js + Express + PostgreSQL).

## Tasks

- [ ] 1. Foundation: Database Schema & Encryption Service
  - [ ] 1.1 Create audit logging table and security configuration
    - Add `security_audit_logs` table to schema.ts with fields: action, actor_id, resource_type, resource_id, status, details, ip_address, user_agent, timestamp, severity
    - Add indexes for efficient queries (timestamp, actor_id, action, resource_type)
    - Add types/migrations for audit log schema
    - _Requirements: 11 (Audit logging)_

  - [ ] 1.2 Implement AES-256-GCM encryption service
    - Create `src/services/encryption.service.ts` with encrypt/decrypt functions
    - Implement IV generation, GCM authentication tags, key rotation support
    - Support optional additional authenticated data (AAD)
    - Write unit tests for round-trip encryption properties
    - _Requirements: 5 (Encryption at rest), 4 (Correctness property: encryption round-trip)_

  - [ ] 1.3 Set up secrets management configuration
    - Create `src/config/secrets.ts` with support for AWS KMS, Vault, and environment variables
    - Implement secret retrieval and caching logic
    - Add secret rotation scheduling hooks
    - Document emergency secret rotation procedure
    - _Requirements: 7 (Secrets management)_

  - [ ] 1.4 Create database migration for security features
    - Add `security_audit_logs` table
    - Add `webauthn_credentials` table for FIDO2 support
    - Add `csrf_tokens` table (optional, if using database-backed tokens)
    - Update users table with `webauthn_enabled` and `backup_codes_encrypted` fields
    - _Requirements: 5 (Encryption at rest), 6 (Encryption in transit), 11 (Audit logging)_

- [ ] 2. CSRF Protection Implementation
  - [ ] 2.1 Implement CSRF token generation and validation middleware
    - Create `src/middleware/csrf.middleware.ts` with generateCSRFToken() and validateCSRFToken()
    - Use cryptographically random nonce generation
    - Implement double-submit cookie pattern with SameSite=Lax
    - Token expiration: 1 hour
    - Support token rotation on each request
    - _Requirements: 1 (CSRF protection), 12 (Security headers - SameSite)_

  - [ ] 2.2 Integrate CSRF middleware into Express app
    - Update `src/index.ts` to apply CSRF middleware to all routes
    - Configure CSRF for state-changing operations (POST, PUT, DELETE)
    - Set SameSite cookie attributes (Lax for most, Strict for admin)
    - Add CSRF error handling to error middleware
    - _Requirements: 1 (CSRF token validation middleware)_

  - [ ]* 2.3 Write CSRF protection tests
    - Write unit tests: valid token accepted, expired token rejected, token rotation
    - Write property test: double-submit consistency property
    - Test CSRF token tamper detection
    - Test cross-site request rejection
    - _Requirements: 1 (CSRF protection tested)_

- [ ] 3. JWT Token Lifecycle & Refresh Mechanism
  - [ ] 3.1 Implement refresh token lifecycle with rotation
    - Extend `src/services/auth.service.ts` with refresh token rotation logic
    - Create `refresh_token_issuances` table queries for token tracking
    - Implement token hash storage (SHA256) for security
    - Add token reuse detection: if old token used after new issued = alert
    - Implement token revocation on logout/password change
    - _Requirements: 2 (Refresh token TTL 7 days, refresh token rotation)_

  - [ ] 3.2 Update token verification to check session/password versions
    - Modify `src/middleware/auth.middleware.ts` to verify session_version and password_version
    - Ensure session version changes invalidate all active tokens instantly
    - Ensure password version changes invalidate tokens when password is changed
    - Test token invalidation after logout
    - _Requirements: 2 (Token binding to session state), 1 (Access token TTL 15 minutes)_

  - [ ] 3.3 Create refresh token endpoint with security validations
    - Implement POST `/auth/refresh` endpoint
    - Validate refresh token signature, expiration, session version
    - Issue new access + refresh token pair
    - Mark old refresh token as revoked (token reuse detection)
    - Delete expired refresh tokens (cron cleanup)
    - _Requirements: 2 (Refresh token rotation, refresh token revocation)_

  - [ ]* 3.4 Write token lifecycle property and unit tests
    - Write property test: token rotation idempotence (new token != old token)
    - Write unit test: access token expires after 15 minutes
    - Write unit test: refresh token expires after 7 days
    - Write unit test: token reuse detected and logged
    - Write unit test: session invalidation revokes all tokens
    - _Requirements: 2 (Token binding, token rotation), 4 (Correctness property: token rotation)_

- [ ] 4. Role-Based Access Control (RBAC) & Permission Matrix
  - [ ] 4.1 Define RBAC roles and permission matrix
    - Create `src/config/rbac.ts` with Role enum and Permission enum
    - Define PERMISSION_MATRIX mapping roles to permissions
    - Roles: user, admin, oracle, moderator
    - Permissions: market operations, user operations, admin operations, oracle operations, moderation
    - Include resource-level permission logic (user can only access own profile)
    - _Requirements: 4 (Permission matrix, least privilege enforcement)_

  - [ ] 4.2 Implement RBAC middleware and permission checks
    - Create `src/middleware/rbac.middleware.ts` with requirePermission() middleware
    - Implement checkResourcePermission() function for resource-level access
    - Add permission denial audit logging
    - Support both role-based and resource-based checks
    - _Requirements: 4 (Permissions checked on every endpoint), 11 (Permission denials logged)_

  - [ ] 4.3 Apply RBAC to protected endpoints
    - Update all protected routes to use requirePermission() middleware
    - Apply to admin routes (DELETE /api/admin/users/:id)
    - Apply to user routes (GET /api/users/:id - own profile only)
    - Apply to market routes (POST /api/markets - admin only)
    - Document required permissions for each endpoint
    - _Requirements: 4 (Resource-level permissions)_

  - [ ]* 4.4 Write RBAC permission tests
    - Write property test: permission matrix enforcement for all role/action/resource combinations
    - Write unit test: user cannot access other user's profile
    - Write unit test: admin can access any resource
    - Write unit test: permission denials logged to audit
    - Write unit test: oracle can submit reports, user cannot
    - _Requirements: 4 (Permission checks), 11 (Audit logging), 4 (Correctness property: RBAC)_

- [ ] 5. Multi-Factor Authentication Hardening (2FA)
  - [ ] 5.1 Encrypt TOTP secrets and implement backup codes
    - Update 2FA secret storage: encrypt with AES-256-GCM before storing
    - Create backup codes generation (8 codes, 10 chars each)
    - Store backup codes encrypted in database
    - Implement backup code validation and one-time use
    - Add 2FA recovery documentation
    - _Requirements: 3 (2FA hardening), 5 (Encryption at rest - 2FA secrets)_

  - [ ] 5.2 Add WebAuthn/FIDO2 support (hardware keys)
    - Implement WebAuthn registration endpoint: POST `/auth/webauthn/register`
    - Implement WebAuthn authentication endpoint: POST `/auth/webauthn/authenticate`
    - Store WebAuthn credentials in `webauthn_credentials` table
    - Support multiple credentials per user (backup keys)
    - Use `@webauthn/server` npm package for protocol implementation
    - _Requirements: 3 (WebAuthn support added), 12 (Input validation for WebAuthn challenges)_

  - [ ] 5.3 Enforce 2FA for admin accounts
    - Add `two_factor_required` flag to user schema (false for users, true for admins)
    - Block admin login if 2FA not completed
    - Add endpoint to download 2FA QR code
    - Add endpoint to download backup codes
    - _Requirements: 3 (2FA mandatory for admin accounts)_

  - [ ]* 5.4 Write 2FA security tests
    - Write unit test: 2FA secret encrypted before storage
    - Write unit test: backup codes can only be used once
    - Write unit test: WebAuthn challenges are cryptographically random
    - Write unit test: WebAuthn challenge can only be used once
    - Write unit test: admin cannot login without 2FA
    - _Requirements: 3 (2FA hardened, WebAuthn support, admin 2FA mandatory)_

- [ ] 6. Input Validation & Output Encoding
  - [ ] 6.1 Implement comprehensive input validation with Zod schemas
    - Create `src/validators/user.validator.ts` with CreateUserSchema, LoginSchema, etc.
    - Create `src/validators/market.validator.ts` with MarketSchema, BetSchema
    - Create `src/validators/admin.validator.ts` with AdminActionSchema
    - Enforce strong password requirements: 12+ chars, uppercase, digit, special char
    - Validate email format, username alphanumeric only
    - Validate numeric inputs (bet amounts, odds)
    - _Requirements: 8 (Input validation with Zod), 9 (Content-type validation)_

  - [ ] 6.2 Create validation middleware and apply to routes
    - Create `src/middleware/validation.middleware.ts` with validateRequest() middleware
    - Apply validation middleware to all user-input endpoints
    - Never expose validation details in error responses (generic error message)
    - Log validation failures for debugging (internal logging only)
    - _Requirements: 8 (Validation implemented), 10 (Stack traces not returned)_

  - [ ] 6.3 Implement output encoding and response validation
    - Create HTML encoding function (escape &, <, >, ", ')
    - Create response validation function for Zod schemas
    - Apply to all JSON responses to prevent XSS
    - Implement timing-safe token comparison to prevent timing attacks
    - _Requirements: 8 (Output encoding), 10 (Timing attacks prevented)_

  - [ ] 6.4 Add file upload validation if applicable
    - If file uploads exist: create file upload validator
    - Check file size (max 5MB)
    - Verify MIME type matches file signature (magic bytes)
    - Scan for malware (if enabled)
    - _Requirements: 8 (File upload validation)_

  - [ ]* 6.5 Write input validation tests
    - Write unit test: invalid email rejected
    - Write unit test: weak password rejected
    - Write unit test: SQL injection payload escaped/rejected
    - Write unit test: XSS payload HTML-encoded
    - Write unit test: oversized input rejected (bet amount > max)
    - _Requirements: 8 (Input validation tested)_

- [ ] 7. Encryption in Transit (HTTPS/TLS 1.3, Security Headers)
  - [ ] 7.1 Configure HTTPS enforcement and TLS 1.3
    - Update Express app to enforce HTTP → HTTPS redirect
    - Verify TLS 1.3 minimum in nginx/load balancer config
    - Disable TLS 1.0, 1.1, 1.2
    - Configure modern cipher suites (TLS_AES_256_GCM_SHA384, etc.)
    - _Requirements: 6 (HTTPS enforced, TLS 1.3 minimum)_

  - [ ] 7.2 Implement security headers
    - Add middleware to set all security headers on every response
    - Headers: X-Frame-Options, X-Content-Type-Options, CSP, Referrer-Policy, Permissions-Policy
    - HSTS: max-age=31536000, includeSubDomains, preload
    - CSP: restrictive default-src 'self'
    - _Requirements: 12 (Security headers configured)_

  - [ ] 7.3 Configure WebSocket Secure (WSS)
    - Ensure WebSocket connections use WSS (ws:// → wss://)
    - Use same TLS certificate as HTTPS
    - Verify existing WebSocket code uses WSS
    - _Requirements: 6 (WSS enforced for WebSocket connections)_

  - [ ]* 7.4 Test security headers and HTTPS configuration
    - Write integration test: HTTP request redirects to HTTPS
    - Write integration test: TLS 1.3 negotiated
    - Write integration test: security headers present in response
    - Write integration test: CSP policy enforced
    - _Requirements: 6 (Encryption in transit enforced), 12 (Security headers)_

- [ ] 8. Rate Limiting & Brute Force Protection
  - [ ] 8.1 Enhance rate limiting with brute force detection
    - Update existing `src/middleware/rate-limit.middleware.ts` with brute force logic
    - Implement progressive backoff: 1st fail = 1s delay, 2nd = 2s, etc.
    - Implement account lockout: 5 failed attempts in 15 minutes = 15 min lockout
    - Implement distributed attack detection: 10+ IPs attempting same resource
    - _Requirements: 9 (Rate limiting, brute force protection, account lockout)_

  - [ ] 8.2 Add CAPTCHA on repeated failures (optional)
    - Integrate hCaptcha or Google reCAPTCHA for login
    - Trigger CAPTCHA after 3 failed login attempts
    - Verify CAPTCHA token server-side before accepting login
    - _Requirements: 9 (CAPTCHA on repeated failures)_

  - [ ] 8.3 Create alerts for suspicious activity
    - Add monitoring for brute force attempts
    - Alert on: distributed attacks, account lockouts, unusual patterns
    - Log suspicious activity to audit log with 'critical' severity
    - _Requirements: 9 (Alerts on suspicious activity), 11 (Audit logging)_

  - [ ]* 8.4 Write rate limiting and brute force tests
    - Write unit test: 6th login attempt rejected
    - Write unit test: account locked after 5 failures
    - Write unit test: account unlocked after 15 minutes
    - Write unit test: distributed attack detected (10+ IPs)
    - _Requirements: 9 (Rate limiting tested)_

- [ ] 9. Audit Logging Implementation
  - [ ] 9.1 Implement core audit logging service
    - Create `src/services/audit.service.ts` with auditLog() function
    - Support all AuditAction types: LOGIN, LOGOUT, PERMISSION_DENIED, SECRET_ROTATED, etc.
    - Capture actor (user/service), resource, action, status, details, IP, user-agent
    - Determine severity level (info/warning/critical) based on action
    - _Requirements: 11 (Audit logging for all sensitive actions)_

  - [ ] 9.2 Audit authentication events
    - Log on successful login: actor_id, timestamp, ip_address, user_agent
    - Log on failed login: actor_id (or email), reason, ip_address
    - Log on logout: actor_id, timestamp
    - Log on password change: actor_id, timestamp
    - Log on 2FA enable/disable: actor_id, timestamp
    - _Requirements: 11 (Authentication events logged)_

  - [ ] 9.3 Audit authorization events
    - Log permission denials: actor_id, resource, permission, timestamp
    - Log role changes: admin_id, target_user_id, old_role, new_role, timestamp
    - Log resource access failures
    - _Requirements: 11 (Authorization events logged)_

  - [ ] 9.4 Audit admin and sensitive actions
    - Log admin actions: user_id, action, resource, timestamp, details
    - Log data modifications: who changed what when
    - Log secret rotations: SYSTEM, secret_name, old_version, new_version
    - Log key rotations: SYSTEM, key_id, timestamp
    - _Requirements: 11 (Admin actions logged with context)_

  - [ ] 9.5 Create audit log queries and investigation tools
    - Implement getAuditTrail(resourceType, resourceId) for resource-specific audit trails
    - Implement getAuditByActor(userId) for user activity investigation
    - Implement getAuditByAction(action) for pattern detection
    - Add pagination and filtering
    - Create admin endpoint GET `/api/admin/audit-logs` with permission checks
    - _Requirements: 11 (Audit logs queryable for investigation)_

  - [ ]* 9.6 Write audit logging tests
    - Write unit test: login successful creates audit log entry
    - Write unit test: permission denial creates audit log entry
    - Write unit test: audit log entry immutable (data not modified on read)
    - Write property test: immutability property for audit logs
    - Write unit test: admin can query audit logs, user cannot
    - _Requirements: 11 (Audit logging tested), 4 (Correctness property: audit log immutability)_

- [ ] 10. Secrets Management & Configuration
  - [ ] 10.1 Implement secret rotation scheduling
    - Create `src/cron/secret-rotation.cron.ts` cron job
    - Run quarterly (90 days) for automated rotation
    - Support manual rotation trigger (admin endpoint)
    - Generate new secrets using cryptographically secure randomness
    - _Requirements: 7 (Secrets rotated on schedule)_

  - [ ] 10.2 Integrate with secrets vault (AWS KMS or HashiCorp Vault)
    - Update `src/config/secrets.ts` to support vault providers
    - Implement secret versioning for rotation (keep last 3 versions)
    - Implement secret retrieval with caching and TTL
    - Document emergency secret rotation procedure
    - _Requirements: 7 (Secrets in vault, not environment variables)_

  - [ ] 10.3 Add secret access auditing
    - Log when secrets are retrieved: service_name, secret_name, timestamp
    - Log when secrets are rotated: old_version, new_version, rotated_by
    - Add alerts for unusual secret access patterns
    - _Requirements: 7 (Access to secrets audited)_

  - [ ]* 10.4 Test secrets management
    - Write unit test: secrets loaded from vault on startup
    - Write unit test: secret rotation generates new secret
    - Write unit test: secret access logged to audit
    - Write integration test: application continues if vault temporarily unavailable
    - _Requirements: 7 (Secrets management tested)_

- [ ] 11. Checkpoint - All Security Components Integrated
  - Ensure all tests pass (unit, property, integration)
  - Verify no security-related errors in logs
  - Check that audit logs contain expected entries for all actions
  - Confirm HTTPS/TLS working correctly
  - Ask the user if they have questions or concerns before moving to testing phase

- [ ] 12. Security Headers & API Response Hardening
  - [ ] 12.1 Implement comprehensive security headers middleware
    - Create `src/middleware/security-headers.middleware.ts`
    - Set all required headers: X-Frame-Options, X-Content-Type-Options, CSP, HSTS, Referrer-Policy, Permissions-Policy
    - Configure CSP policy (restrictive by default)
    - Apply to all responses
    - _Requirements: 12 (Security headers configured)_

  - [ ] 12.2 Implement secure error responses
    - Update error middleware to never expose stack traces in production
    - Generalize database errors (no SQL details)
    - Use generic error messages (no internal details)
    - Log full error details internally for debugging
    - _Requirements: 10 (Error handling secure)_

  - [ ] 12.3 Prevent information disclosure
    - Remove or mask version headers (X-Powered-By, Server)
    - Implement constant-time token comparison
    - Avoid timing leaks in authentication logic
    - _Requirements: 10 (Information disclosure prevented)_

  - [ ]* 12.4 Test security headers and error handling
    - Write integration test: CSP header present and valid
    - Write integration test: stack trace not in error response (production mode)
    - Write unit test: database error generalized to generic message
    - Write unit test: constant-time comparison prevents timing attacks
    - _Requirements: 12 (Security headers tested), 10 (Error handling tested)_

- [ ] 13. Final Checkpoint & Verification
  - Run full test suite (all unit, property, integration, security tests pass)
  - Verify no TODO comments or placeholder code remaining
  - Confirm audit logging working end-to-end
  - Verify all endpoints have appropriate permission checks
  - Ensure all dependencies are up-to-date and security-patched
  - Ask the user if they have questions or concerns

## Implementation Notes

### Key Files to Create
- `src/services/encryption.service.ts` - AES-256-GCM encryption/decryption
- `src/services/audit.service.ts` - Audit log writing and querying
- `src/config/rbac.ts` - Role definitions and permission matrix
- `src/config/secrets.ts` - Secrets management configuration
- `src/middleware/csrf.middleware.ts` - CSRF token generation/validation
- `src/middleware/rbac.middleware.ts` - RBAC enforcement
- `src/middleware/security-headers.middleware.ts` - Security headers
- `src/validators/*.ts` - Zod schemas for input validation

### Key Files to Update
- `src/services/auth.service.ts` - Token lifecycle, refresh rotation
- `src/middleware/auth.middleware.ts` - Token verification enhancements
- `src/middleware/rate-limit.middleware.ts` - Brute force detection
- `src/middleware/error.middleware.ts` - Secure error responses
- `src/index.ts` - Apply CSRF, RBAC, security headers middleware
- `src/db/schema.ts` - Add audit log, WebAuthn, backup codes tables
- All route files - Apply permission checks, input validation

### Dependencies to Add
```json
{
  "@webauthn/server": "^0.15.0",
  "@webauthn/browser": "^0.15.0"
}
```

### Execution Order Matters
1. Foundation (tables, encryption, secrets) must be done first
2. CSRF and token lifecycle can be done in parallel with RBAC setup
3. Input validation should be applied to all routes after RBAC
4. Rate limiting enhancements depend on rate-limit middleware existing
5. Audit logging should be integrated throughout other components
6. Security headers can be added at the end

### Testing Strategy
- **Property tests**: CSRF double-submit, token rotation, RBAC matrix, encryption round-trip, audit immutability
- **Unit tests**: Specific examples, edge cases, error conditions (5-10 per task)
- **Integration tests**: Database persistence, vault integration, HTTPS/TLS, WebAuthn
- Mark test tasks with `*` to indicate they're optional but recommended

### Security Considerations
- Never commit secrets to Git (verified by gitleaks in CI)
- Use HttpOnly, Secure, SameSite flags for all cookies
- All passwords hashed with bcrypt (12 rounds)
- All tokens signed with strong algorithms (HS256 minimum)
- All encryption keys rotated quarterly
- All sensitive operations logged to audit
- All errors sanitized (no internals exposed)
