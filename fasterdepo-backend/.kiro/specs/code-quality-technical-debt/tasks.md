# Implementation Plan: Code Quality & Technical Debt

## Overview

This plan addresses technical debt through four main phases: schema consolidation, Docker configuration cleanup, service refactoring, and test infrastructure integration. Each phase builds on the previous one, enabling incremental progress and verification.

---

## Tasks

- [ ] 1. Schema Consolidation
  - [ ] 1.1 Analyze duplicate table definitions in schema.ts
    - Identify all duplicate definitions (user_streaks, achievements, user_achievements, referrals)
    - Document differences between first and second definitions
    - Create mapping of which definition to keep/rename
    - _Requirements: Fix duplicate table definitions in schema.ts_
  
  - [ ] 1.2 Create migration to rename engagement tables
    - Generate migration file with `pg_rename_table` operations
    - Rename first-occurrence tables: `user_streaks` → `engagement_user_streaks`, `achievements` → `engagement_achievements`, `referrals` → `engagement_referrals`
    - Include data migration logic if needed
    - Verify migration is reversible
    - _Requirements: Fix duplicate table definitions in schema.ts_
  
  - [ ] 1.3 Update schema.ts with consolidated definitions
    - Remove first occurrence of user_streaks, achievements, user_achievements, referrals
    - Rename and relocate engagement tables
    - Keep second-occurrence user_id based definitions as primary
    - Update all TypeScript type exports
    - _Requirements: Fix duplicate table definitions in schema.ts_
  
  - [ ] 1.4 Update all service references to use correct tables
    - Search for references to user_streaks, achievements, user_achievements, referrals across services
    - Update imports and queries to use new engagement table names where needed
    - Fix any type mismatches caused by schema changes
    - _Requirements: Fix duplicate table definitions in schema.ts_
  
  - [ ]* 1.5 Write integration tests for schema consolidation
    - Test that migrations run successfully
    - Test that both engagement and primary tables work correctly
    - Verify foreign key constraints are enforced
    - Test round-trip migrations (up and down)
    - _Requirements: Fix duplicate table definitions in schema.ts_
  
  - [ ] 1.6 Checkpoint - Run TypeScript compiler
    - Execute `npm run build` to verify schema changes compile
    - Fix any type errors that arise from schema consolidation
    - Run linter to check code style
    - _Requirements: Fix TypeScript compilation errors_

- [ ] 2. Docker Configuration & Middleware Audit
  - [ ] 2.1 Fix docker-compose.yml duplicate indexer service
    - Identify and remove second indexer service definition
    - Consolidate configuration into single indexer service
    - Verify indexer depends on both postgres and backend
    - Add environment overrides for FACTORY_CONTRACT_ADDRESS and SOROBAN_RPC_URL
    - _Requirements: Fix duplicate indexer service in docker-compose.yml_
  
  - [ ] 2.2 Audit all admin endpoints for middleware protection
    - Review AdminController.ts and all admin route handlers
    - Verify each admin endpoint has `requireAdmin` or `requireAdminJwt` middleware
    - Document any missing middleware
    - Create list of endpoints requiring fixes
    - _Requirements: Implement missing requireAdminJwt middleware, Ensure consistent error handling_
  
  - [ ] 2.3 Add missing middleware to unprotected admin endpoints
    - Apply `requireAdmin` middleware to all user-initiated admin routes
    - Apply `requireAdminJwt` middleware to service-to-service admin operations
    - Update route definitions in all controllers
    - _Requirements: Implement missing requireAdminJwt middleware_
  
  - [ ]* 2.4 Write integration tests for middleware
    - Test that requireAdmin blocks unauthenticated requests
    - Test that requireAdmin blocks non-admin users
    - Test that requireAdminJwt validates service tokens
    - Test that requireAdmin checks session revocation
    - _Requirements: Implement missing requireAdminJwt middleware_
  
  - [ ] 2.5 Checkpoint - Docker and middleware verification
    - Run `docker-compose config` to verify no conflicts
    - Verify all services start without errors: `docker-compose up -d`
    - Test admin endpoints with and without valid tokens
    - Verify error responses follow standard format
    - _Requirements: Fix duplicate indexer service, Implement requireAdminJwt_

- [ ] 3. Error Handling Standardization
  - [ ] 3.1 Audit all controllers for consistent error handling
    - Review each controller file (AdminController, BetController, MarketController, etc.)
    - Check that all error paths use AppError class
    - Verify no try-catch blocks swallow errors silently
    - Document any controllers not following patterns
    - _Requirements: Ensure consistent error handling across all controllers_
  
  - [ ] 3.2 Update controllers to use standard error handling pattern
    - Wrap all business logic in try-catch
    - Throw AppError with appropriate status, message, and code
    - Let errorMiddleware handle error responses
    - Remove any custom error response formatting
    - _Requirements: Ensure consistent error handling across all controllers_
  
  - [ ] 3.3 Add error codes to errorCodes.ts
    - Review all unique error scenarios in controllers
    - Add error code constant for each scenario
    - Update controllers to use error code constants
    - Document error codes in constants file
    - _Requirements: Ensure consistent error handling across all controllers_
  
  - [ ] 3.4 Verify error response format consistency
    - Test each error scenario (400, 401, 403, 404, 422, 500)
    - Verify response includes statusCode, message, and code
    - Verify no stack traces appear in responses
    - Verify logs include full error context (for debugging)
    - _Requirements: Ensure consistent error handling across all controllers_
  
  - [ ]* 3.5 Write error handling tests
    - Test each error code returns correct status
    - Test missing required fields return 400
    - Test invalid tokens return 401
    - Test insufficient permissions return 403
    - Test not found scenarios return 404
    - _Requirements: Ensure consistent error handling across all controllers_
  
  - [ ] 3.6 Checkpoint - Error handling verification
    - Run test suite to verify error handling works
    - Manually test API with invalid inputs
    - Verify error logs are informative for debugging
    - Check response format matches specification
    - _Requirements: Ensure consistent error handling across all controllers_

- [ ] 4. Service Refactoring & Test Integration
  - [ ] 4.1 Refactor MarketService into smaller services
    - Extract MarketOddsService from MarketService (odds calculation methods)
    - Extract MarketAnalyticsService from MarketService (analytics methods)
    - Keep MarketService focused on core market operations
    - Update service composition so MarketService uses new services
    - Verify no logic duplication across services
    - _Requirements: Refactor large services (MarketService > 500 lines)_
  
  - [ ] 4.2 Refactor BetService into smaller services
    - Extract BetPayoutService from BetService (payout logic)
    - Extract BetValidationService from BetService (validation logic)
    - Keep BetService focused on core bet operations
    - Update service composition so BetService uses new services
    - Verify no logic duplication across services
    - _Requirements: Refactor large services (BetService > 500 lines)_
  
  - [ ] 4.3 Update controllers to use refactored services
    - Update MarketController to use new market services
    - Update BetController to use new bet services
    - Verify all endpoint functionality works correctly
    - Test that controller logic is unchanged (refactoring only)
    - _Requirements: Refactor large services_
  
  - [ ]* 4.4 Write unit tests for refactored services
    - Write tests for MarketOddsService odds calculations
    - Write tests for BetPayoutService payout logic
    - Write tests for BetValidationService validation rules
    - Write tests for MarketAnalyticsService statistics
    - Mock database calls to test in isolation
    - _Requirements: Refactor large services_
  
  - [ ] 4.5 Integrate e2e-cleanup into test suite
    - Update test setup to run cleanup before tests
    - Update test teardown to run cleanup after tests
    - Extend cleanup script to handle new tables (user_sessions, refresh_tokens, etc.)
    - Add cleanup for engagement tables if test data created
    - Verify cleanup properly handles transactions
    - _Requirements: Integrate test cleanup (e2e-cleanup.js)_
  
  - [ ]* 4.6 Write cleanup integration tests
    - Test that cleanup removes test-prefixed data
    - Test that cleanup preserves production data
    - Test that cleanup handles missing tables gracefully
    - Test that cleanup is idempotent (safe to run multiple times)
    - Test cleanup in transaction (rollback on failure)
    - _Requirements: Integrate test cleanup (e2e-cleanup.js)_
  
  - [ ] 4.7 Checkpoint - Service and test verification
    - Run all tests with new service structure: `npm test`
    - Verify TypeScript compilation: `npm run build`
    - Run integration tests: `npm run test:integration`
    - Check code coverage is maintained
    - Verify no performance regressions
    - _Requirements: Refactor services, Integrate cleanup_
  
  - [ ] 4.8 Code quality verification
    - Run linter: `npm run lint` 
    - Check for circular dependencies
    - Verify all services follow max 300 line limit
    - Check error handling in new services
    - Verify test coverage for refactored code
    - _Requirements: Fix TypeScript compilation errors, Ensure consistent error handling_

- [ ] 5. Final Integration & Deployment
  - [ ] 5.1 Full system test
    - Start services: `docker-compose up -d`
    - Run full test suite: `npm test`
    - Test key workflows: create market → place bet → resolve
    - Verify admin endpoints require auth
    - Check error responses are formatted correctly
    - _Requirements: All previous requirements_
  
  - [ ] 5.2 Documentation updates
    - Update CONTRIBUTING.md with error handling pattern
    - Document service responsibilities and relationships
    - Add schema consolidation notes to deployment guide
    - Document Docker Compose configuration
    - Document cleanup script usage
    - _Requirements: All previous requirements_
  
  - [ ] 5.3 Final checkpoint - All tests pass
    - Run full test suite: `npm run test:full`
    - Verify no compilation errors: `npm run build`
    - Verify linting passes: `npm run lint`
    - Verify Docker services start: `docker-compose up -d`
    - All checkpoint tasks pass
    - _Requirements: All previous requirements_

---

## Success Criteria

- ✅ No duplicate table definitions in schema.ts
- ✅ No duplicate service definitions in docker-compose.yml
- ✅ All admin endpoints protected with middleware
- ✅ All controllers follow consistent error handling pattern
- ✅ No service exceeds 300 lines
- ✅ TypeScript compilation succeeds without errors
- ✅ All tests pass
- ✅ Error responses follow standard format
- ✅ E2E cleanup properly integrated

---
