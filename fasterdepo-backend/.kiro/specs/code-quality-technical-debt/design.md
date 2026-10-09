# Design Document: Code Quality & Technical Debt

## Overview

This design addresses critical technical debt issues in the BANKERCHANGER backend codebase: duplicate table definitions in the Drizzle schema, Docker configuration duplication, missing middleware implementation, inconsistent error handling, and oversized service modules. The solution provides a consolidated, maintainable architecture that improves code quality and developer experience.

---

## 1. Schema Consolidation Strategy

### Problem Analysis

The `src/db/schema.ts` file contains four tables defined twice with conflicting schemas:
- **user_streaks**: First definition (lines ~380-390) vs second definition (lines ~580-600)
- **achievements**: First definition (lines ~390-410) vs second definition (lines ~600-620)
- **user_achievements**: First definition (lines ~410-430) vs second definition (lines ~620-640)
- **referrals**: First definition (lines ~430-450) vs second definition (lines ~640-660)

**Root Cause**: Engagement features were added in multiple phases without proper de-duplication. The first definitions use `address` fields (Stellar account identifiers), while the second definitions use `user_id` fields (database user records).

### Resolution Strategy

**Decision**: Use the second definitions (`user_id` based) as the canonical versions because:
1. They properly reference the `users` table with cascading deletes
2. They align with the session/authentication system (users have IDs)
3. They enable proper audit trails and user management
4. The first definitions using `address` fields are kept as engagement metrics tables

**Implementation Approach**:
1. **Remove duplicate definitions** (first occurrences of user_streaks, achievements, user_achievements, referrals)
2. **Rename first-occurrence tables** to indicate their purpose:
   - `user_streaks` (address-based) → `engagement_user_streaks` (metrics for Stellar addresses)
   - `achievements` (first) → `engagement_achievements` (platform-wide achievement definitions)
   - `referrals` (first) → `engagement_referrals` (address-based referral tracking)
3. **Keep second-occurrence tables** as primary (`user_streaks`, `achievements`, `user_achievements`, `referrals`) with `user_id` references
4. **Create migration** to rename tables and handle data migration if needed
5. **Update all imports and references** across services

### Schema Structure After Consolidation

**Authentication & User Management**:
- `users` - Core user records
- `user_sessions` - Active sessions
- `password_reset_tokens` - Password reset workflow
- `refresh_token_issuances` - Token lifecycle management

**Engagement (Address-Based)**:
- `engagement_user_streaks` - Streak tracking by Stellar address
- `engagement_achievements` - Achievement definitions
- `engagement_referrals` - Referral tracking by address (legacy/transition)

**User Management (ID-Based)**:
- `user_streaks` - Streak tracking by user ID
- `achievements` - Achievement definitions
- `user_achievements` - User achievement awards
- `referrals` - Referral relationships
- `referral_payouts` - Referral compensation

### Validation Rules

Each table definition MUST have:
1. ✅ Unique primary key
2. ✅ No duplicate column definitions
3. ✅ Proper foreign key constraints with cascading deletes where appropriate
4. ✅ Appropriate indexes for query patterns
5. ✅ Timestamps (created_at, updated_at where applicable)
6. ✅ Type exports for TypeScript safety

---

## 2. Docker Configuration Fixes

### Problem Analysis

The `docker-compose.yml` file defines the `indexer` service twice (lines 35-40 and 51-61):
- **First definition**: Depends on postgres, references `indexer/.env`
- **Second definition**: Depends on backend, references `backend/.env`

**Root Cause**: Service configuration was amended during development without removing the original definition.

### Resolution Strategy

**Decision**: Consolidate into a single `indexer` service definition with proper dependencies.

**Corrected Configuration**:
```yaml
indexer:
  build: ./indexer
  restart: unless-stopped
  depends_on:
    postgres:
      condition: service_healthy
    backend:
      condition: service_started
  env_file: ./indexer/.env
  environment:
    - FACTORY_CONTRACT_ADDRESS=${FACTORY_CONTRACT_ADDRESS}
    - SOROBAN_RPC_URL=${STELLAR_RPC_URL:-https://soroban-testnet.stellar.org}
  ports:
    - '3002:3002'
  volumes:
    - indexer_data:/app/data
```

**Key Changes**:
1. Single service definition (remove second one)
2. Depends on both postgres and backend (wait for both to be ready)
3. Uses `indexer/.env` for base configuration
4. Environment overrides for blockchain parameters
5. Proper volume management for data persistence

### Validation

- ✅ No duplicate service definitions
- ✅ Correct dependency ordering (postgres → indexer → frontend, or postgres+backend → indexer)
- ✅ All required environment variables set
- ✅ Service can restart without conflicts

---

## 3. Middleware Implementation Details

### Current State

The `requireAdminJwt` middleware exists and is properly implemented with:
- Service-to-service JWT verification using `ADMIN_JWT_SECRET`
- User admin access verification using `JWT_ACCESS_SECRET`
- Session revocation checks via Redis
- Proper error handling with `AppError`

**Issue**: The middleware is correctly implemented but may not be consistently applied across all admin endpoints.

### Verification Strategy

The design approach is to:
1. **Audit all admin endpoints** (AdminController, etc.) to ensure `requireAdminJwt` or `requireAdmin` is applied
2. **Establish consistent patterns** for admin route protection
3. **Document middleware application rules**

### Middleware Application Rules

**Rule 1: Service-to-Service Admin Operations**
- Use: `requireAdminJwt` middleware
- Example: Automated admin tasks, cron jobs
- JWT signed with: `ADMIN_JWT_SECRET`

**Rule 2: User Admin Operations**
- Use: `requireAdmin` middleware  
- Example: Dashboard admin routes, user-initiated admin actions
- JWT signed with: `JWT_ACCESS_SECRET` (user session)
- Checks: Token type = 'access', role = 'admin', session not revoked

**Rule 3: Route Protection Pattern**
```typescript
router.post('/admin/disputes/:id/resolve', 
  requireAdmin,  // Verify user is admin
  validateRequest(schema),  // Validate input
  handleAsync(resolveDispute)  // Handler
);
```

---

## 4. Error Handling Patterns to Enforce

### Current State

The error handling middleware is well-designed:
- Centralized error handling via `errorMiddleware`
- Custom `AppError` class with status codes and error codes
- No stack traces in responses (security)
- Structured error response format
- Separate logging for 5xx errors

### Error Handling Standard Pattern

**All controllers MUST follow this pattern**:

```typescript
export async function handleAction(req: Request, res: Response): Promise<void> {
  try {
    // 1. Validate input (handled by middleware)
    const { id } = req.params;
    
    // 2. Perform business logic
    const result = await service.action(id);
    
    // 3. Return response
    res.json({ 
      success: true, 
      data: result 
    });
    
  } catch (err) {
    // 4. Let errorMiddleware handle it - DO NOT handle here
    throw err instanceof AppError ? err : new AppError(500, 'Unexpected error');
  }
}
```

### Error Code Standard

All errors MUST use the `AppError` class with:
- **statusCode**: HTTP status code (400, 401, 403, 404, 422, 500)
- **message**: User-friendly message
- **code**: Machine-readable error code (optional, for frontend)
- **details**: Additional context (optional, visible to users)

**Example**:
```typescript
throw new AppError(
  404, 
  'Market not found',
  'MARKET_NOT_FOUND',
  { marketId: id }
);
```

### Error Response Format

All error responses follow this structure:
```json
{
  "error": {
    "statusCode": 404,
    "message": "Market not found",
    "code": "MARKET_NOT_FOUND",
    "details": { "marketId": "market_123" }
  }
}
```

### Common Error Scenarios

| Scenario | Status | Code | Message |
|----------|--------|------|---------|
| Invalid input | 400 | INVALID_REQUEST | Describe which field is invalid |
| Not authenticated | 401 | UNAUTHORIZED | Missing or invalid token |
| Not authorized | 403 | FORBIDDEN | User lacks required role/permission |
| Resource not found | 404 | NOT_FOUND | Specific resource type not found |
| Validation failed | 422 | VALIDATION_ERROR | Describe validation issue |
| Database error | 500 | DATABASE_ERROR | Log full error, return generic message |

---

## 5. Service Refactoring Approach

### Problem Analysis

Services are too large (>500 lines), making them difficult to maintain, test, and reason about:
- **MarketService.ts**: ~950 lines (market queries, odds calculations, portfolio analysis)
- **BetService.ts**: ~700 lines (bet placement, claim logic, payout calculations)

### Refactoring Strategy

**Single Responsibility Principle**: Each service handles one domain concern.

#### MarketService Decomposition

**Current**: All market operations in one service

**After Refactoring**:

1. **MarketService** (Core Market Operations)
   - `getMarkets(filters, pagination)` - List markets
   - `getMarketById(market_id)` - Fetch single market
   - `updateMarketStatus(market_id, status)` - Status transitions
   - `bulkPauseMarkets(marketIds)` - Bulk pause
   - `bulkCancelMarkets(marketIds)` - Bulk cancel
   - **~250 lines**

2. **MarketOddsService** (Odds Calculations)
   - `getMarketOdds(market_id)` - Calculate current odds
   - `calculateOdds(market_id)` - Odds computation logic
   - `calculateOutcomeOdds(market_id, outcome)` - Single outcome odds
   - `computeOutcomeOdds(pools, outcome)` - Helper calculation
   - **~150 lines**

3. **MarketAnalyticsService** (Market Analytics)
   - `getMarketStats(market_id)` - Market statistics
   - `getBettorStats(bettor_address)` - Bettor statistics
   - `getPortfolioByAddress(address)` - Portfolio analysis
   - `simulateProjectedPayout(...)` - Payout simulation
   - `getPlatformStats()` - Platform-wide statistics
   - **~150 lines**

#### BetService Decomposition

**Current**: All bet operations in one service

**After Refactoring**:

1. **BetService** (Core Bet Operations)
   - `placeBet(params)` - Create bet
   - `claimPayout(betId)` - Claim winnings
   - `getBetsByAddress(address)` - Fetch user bets
   - `getBetsByMarket(marketId)` - Fetch market bets
   - **~250 lines**

2. **BetPayoutService** (Payout Logic)
   - `calculatePayout(bet, outcome)` - Payout calculation
   - `processPayout(betId)` - Execute payout
   - `distributeBetTax(market, betAmount)` - Tax distribution
   - `validatePayoutEligibility(bet)` - Eligibility check
   - **~150 lines**

3. **BetValidationService** (Bet Validation)
   - `validateBetPlacement(params)` - Pre-placement validation
   - `validateMarketActive(marketId)` - Market state validation
   - `validateBettorFunds(address, amount)` - Fund availability
   - `validateOddsLock(market)` - Odds lock validation
   - **~100 lines**

### Refactoring Execution Plan

1. **Create new service files** with extracted responsibility
2. **Move methods** from parent service to specialized service
3. **Update service layer** to compose services (e.g., `BetService` calls `BetPayoutService`)
4. **Update controllers** to use appropriate services
5. **Add integration points** for cross-service calls
6. **Write unit tests** for each service in isolation
7. **Verify no logic duplication** across services

### Service Composition Pattern

```typescript
// Example: BetController using multiple services
export async function placeBet(req: Request, res: Response): Promise<void> {
  const { marketId, side, amount } = req.body;
  
  // Validation layer
  await betValidationService.validateBetPlacement({ marketId, side, amount });
  
  // Business logic layer
  const bet = await betService.placeBet({ marketId, side, amount });
  
  // Analytics layer
  await marketAnalyticsService.invalidateCache(marketId);
  
  res.json({ success: true, data: bet });
}
```

---

## 6. Test Infrastructure Improvements

### Current State

The `e2e-cleanup.js` script properly cleans test data:
- Validates E2E_DATA_PREFIX environment variable
- Uses transactions for atomicity
- Cleans referential data in correct order
- Handles missing schema gracefully

### Enhancement Strategy

**Phase 1: Integrate e2e-cleanup into test workflow**
- Call cleanup before test suite runs
- Call cleanup after test suite completes
- Ensure database state isolation

**Phase 2: Extend cleanup coverage**
- Add cleanup for `user_sessions` (test-created sessions)
- Add cleanup for `refresh_token_issuances` (test tokens)
- Add cleanup for `admin_audit_logs` if tests create entries
- Add cleanup for `blockchain_events` (test events)

**Phase 3: Add fixture management**
- Create reusable test fixtures for common entities
- Fixture system for markets, users, bets
- Centralized fixture cleanup

### Cleanup Reliability Rules

- ✅ Cleanup runs in a transaction (all-or-nothing)
- ✅ Foreign key constraints respected (delete order)
- ✅ Idempotent (can run multiple times safely)
- ✅ Validates database schema exists
- ✅ Validates environment configuration
- ✅ Logs cleanup results for debugging

---

## Error Handling

### Error Scenarios

1. **Schema Duplication Issues**
   - Migration failure: Rollback and alert
   - Type mismatch: Type checking catches at compile time
   - Reference errors: Database constraints enforce

2. **Docker Configuration**
   - Service startup failure: Docker Compose reports which service failed
   - Port conflicts: Configure different ports for test/dev/prod
   - Network issues: Service health checks detect connectivity problems

3. **Middleware Issues**
   - Missing authorization header: Return 401
   - Invalid JWT: Return 403
   - Session revoked: Return 401
   - Invalid role: Return 403

4. **Service Refactoring Issues**
   - Incomplete data fetching: Tests catch missing calls
   - Logic regression: Unit tests validate behavior
   - Performance regression: Load tests catch slow queries

5. **Cleanup Failures**
   - Transaction rollback on any error
   - Partial cleanup prevented
   - Error logging enables debugging

### Recovery Strategies

- **Schema**: Manual migration rollback if needed
- **Docker**: Container restart with updated config
- **Middleware**: Verify environment variables, check tokens
- **Services**: Deploy previous version, run tests
- **Cleanup**: Manual database cleanup via SQL

---

## Testing Strategy

### Dual Testing Approach

This feature involves structural and configuration changes that should be verified through:

1. **Unit Tests** (Service Refactoring):
   - Test each refactored service in isolation
   - Mock external dependencies (database, cache)
   - Verify business logic correctness
   - Example: `MarketOddsService` calculates odds correctly regardless of pool values

2. **Integration Tests** (Schema & Docker):
   - Test schema migrations and references
   - Verify Docker services start and communicate
   - Test end-to-end workflows
   - Example: Create market → place bet → verify data structure

3. **Code Quality Checks**:
   - TypeScript compilation (catch type errors)
   - Linting (code standards)
   - Import validation (cyclic dependencies)

### Why Property-Based Testing Does NOT Apply

This feature is primarily about code organization and infrastructure fixes, not algorithmic correctness. Property-based testing is inappropriate because:

- **Schema consolidation**: Configuration change, not logic testing
- **Docker fixes**: Infrastructure setup, not behavior testing  
- **Middleware**: Already implemented correctly, just needs audit
- **Service refactoring**: Code reorganization, logic unchanged
- **Cleanup integration**: SQL cleanup operations, not algorithmic

These issues are better addressed through:
- **Type checking** (TypeScript catches schema inconsistencies)
- **Integration tests** (verify Docker/services work together)
- **Code review** (verify refactoring preserves logic)
- **Smoke tests** (verify system starts correctly)

---

## Implementation Notes

### Backward Compatibility

- Schema changes handled via migration
- No API contract changes
- Service refactoring is internal (controllers unchanged)
- Docker config change is transparent to users

### Dependencies

- Drizzle ORM migration support
- Docker Compose v3.9+
- PostgreSQL 15
- Node.js test framework (Jest)

### Success Criteria

✅ All duplicate table definitions removed  
✅ Docker-compose.yml has single indexer definition  
✅ All admin endpoints protected with middleware  
✅ Error handling consistent across all controllers  
✅ Services under 300 lines each  
✅ E2E cleanup properly integrated  
✅ TypeScript compilation without errors  
✅ All existing tests pass  

---
