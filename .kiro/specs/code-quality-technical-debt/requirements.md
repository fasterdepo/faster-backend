# Code Quality & Technical Debt Elimination

## Overview
Fix critical technical debt in the BANKERCHANGER project that blocks other improvements and reduces code maintainability. This includes schema duplications, Docker configuration errors, incomplete middleware implementations, and inconsistent error handling patterns.

## Business Requirements

### Critical Issues (Must Fix)
1. **Schema Duplication Bug**
   - User story: As a developer, I need a single source of truth for database tables so migrations run without conflicts
   - Acceptance criteria:
     - [ ] Remove duplicate `user_streaks` table definition
     - [ ] Remove duplicate `achievements` table definition
     - [ ] Remove duplicate `user_achievements` table definition
     - [ ] Remove duplicate `referrals` table definition
     - [ ] Ensure remaining definitions are correct and complete
     - [ ] Run migrations without errors

2. **Docker Compose Duplication**
   - User story: As an ops engineer, I need correct Docker configuration so services start reliably
   - Acceptance criteria:
     - [ ] Remove duplicate `indexer` service definition
     - [ ] Verify all services (postgres, redis, indexer, backend, frontend) are defined once
     - [ ] Docker compose up should work without service conflicts

3. **Missing requireAdminJwt Middleware**
   - User story: As a security engineer, I need admin endpoint protection so unauthorized users cannot access admin APIs
   - Acceptance criteria:
     - [ ] Create and export `requireAdminJwt` middleware
     - [ ] Implement proper JWT validation for admin routes
     - [ ] Return 403 FORBIDDEN for non-admin users
     - [ ] Return 401 UNAUTHORIZED for missing/invalid tokens
     - [ ] Apply middleware to all admin endpoints

### High Priority Issues
4. **Error Handling Consistency**
   - User story: As a developer, I need consistent error patterns so error handling is predictable
   - Acceptance criteria:
     - [ ] Audit all controllers for consistent AppError usage
     - [ ] Ensure all errors include machine-readable error codes
     - [ ] All async handlers wrapped in try-catch or wrapped with async error handler
     - [ ] Error responses follow standard format

5. **Type Safety Gaps**
   - User story: As a developer, I need TypeScript type coverage so fewer runtime errors occur
   - Acceptance criteria:
     - [ ] Run `npm run build` with no TypeScript errors
     - [ ] Fix any type casting with `any`
     - [ ] Ensure Drizzle schema types are properly exported

### Medium Priority Issues
6. **Service Layer Complexity**
   - User story: As a developer, I need maintainable services so onboarding new developers is easier
   - Acceptance criteria:
     - [ ] Identify services >500 lines (MarketService, BetService, etc.)
     - [ ] Create plan for refactoring into smaller, focused services
     - [ ] Separate concerns (repository, business logic, external integrations)

7. **Test Cleanup Integration**
   - User story: As a QA engineer, I need reliable test cleanup so test runs are isolated
   - Acceptance criteria:
     - [ ] Integrate e2e-cleanup.js into test teardown
     - [ ] Verify all tests properly clean up database state
     - [ ] E2E tests run without test pollution

## Non-Functional Requirements

1. **Code Quality**
   - Zero duplicate code paths
   - All TypeScript strict mode checks passing
   - ESLint with no errors (warnings acceptable)

2. **Testability**
   - Existing tests continue to pass
   - New test cleanup integration doesn't break existing tests

3. **Documentation**
   - Update comments where code changes significantly
   - Document any patterns that were changed (error handling, middleware usage)

## Acceptance Criteria (Overall)

- [ ] All 3 critical schema duplications removed
- [ ] Docker compose.yml has no duplicate service definitions
- [ ] requireAdminJwt middleware created and applied to admin routes
- [ ] All TypeScript builds pass with no errors
- [ ] All existing tests continue to pass
- [ ] Error handling audit completed with consistency improvements applied
- [ ] Service layer complexity assessment documented with refactoring plan
- [ ] Test cleanup integration working properly

## Success Metrics

1. No schema conflicts on database migrations
2. Docker Compose starts without errors
3. Admin endpoints properly protected (verified with manual testing)
4. TypeScript compilation time < 10 seconds
5. All unit/integration tests passing
6. Code coverage maintained (no decrease from baseline)
