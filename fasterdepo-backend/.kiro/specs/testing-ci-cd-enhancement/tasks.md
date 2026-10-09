# Implementation Plan: Testing & CI/CD Enhancement

## Overview

This implementation plan adds comprehensive testing infrastructure enhancements to BANKERCHANGER backend. The work is organized in phases: establishing foundations (fixtures, utilities), enhancing the CI/CD pipeline with coverage enforcement, implementing security testing, creating E2E critical flow tests, and establishing load testing baselines. Each phase builds on the previous and integrates incrementally.

## Tasks

- [ ] 1. Create test fixtures and data factory framework
  - [ ] 1.1 Create test fixture directory structure and README
    - Create `tests/fixtures/` directory
    - Create `tests/fixtures/README.md` with factory usage guide
    - Document factory patterns and conventions
    - _Requirements: Fixture and mocking strategy, test data factory pattern_
  
  - [ ] 1.2 Implement base factory interface and utilities
    - Create `tests/fixtures/factory.base.ts` with `TestDataFactory<T>` interface
    - Implement `createFactory()` helper for consistent factory creation
    - Add support for `one()`, `batch()`, `invalid()`, `edge()` methods
    - _Requirements: Test data factory pattern, fixture documentation_
  
  - [ ] 1.3 Implement User factory with comprehensive overrides
    - Create `tests/fixtures/user.factory.ts`
    - Support role-based creation (user, admin, oracle)
    - Support invalid/edge cases (unicode, max length, SQL injection payloads)
    - Include password generation with complexity requirements
    - _Requirements: Test data factory pattern, test isolation strategy_
  
  - [ ] 1.4 Implement Market factory with outcome variations
    - Create `tests/fixtures/market.factory.ts`
    - Support binary, ternary, and multi-outcome markets
    - Support various market states (draft, open, locked, resolved)
    - Support invalid edge cases (bad dates, invalid odds)
    - _Requirements: Test data factory pattern_
  
  - [ ] 1.5 Implement Bet factory with various bet types
    - Create `tests/fixtures/bet.factory.ts`
    - Support different bet sizes (valid, edge cases)
    - Support various outcome selections
    - Include correlation with market/user factories
    - _Requirements: Test data factory pattern_

- [ ] 2. Enhance test database setup and isolation
  - [ ] 2.1 Create isolated test database transaction wrapper
    - Create `tests/utils/db-isolation.ts`
    - Implement `testTransaction()` that manages DB transaction lifecycle
    - Add automatic rollback on test completion
    - Verify no state leakage between tests
    - _Requirements: Test isolation strategy, database transaction wrapper_
  
  - [ ] 2.2 Create test database initialization script
    - Create `tests/utils/db-setup.ts`
    - Run migrations against test database
    - Create separate `bankerchanger_test` database if needed
    - Add connection pooling for test suite
    - _Requirements: Test isolation strategy, test database setup_
  
  - [ ] 2.3 Create Redis cache cleanup utilities
    - Create `tests/utils/redis-cleanup.ts`
    - Implement `flushTestRedis()` for cache clearing
    - Add namespace isolation for test data
    - _Requirements: Test isolation strategy, test data management_
  
  - [ ] 2.4 Create cleanup/teardown script for E2E tests
    - Create `tests/e2e/teardown.ts`
    - Implement `cleanupTestData()` that deletes by ID prefix
    - Verify all orphaned resources are removed
    - Add logging for cleanup operations
    - _Requirements: Test isolation strategy, E2E test cleanup_

- [ ] 3. Implement comprehensive security test suite
  - [ ] 3.1 Create SQL injection prevention tests
    - Create `tests/security/sql-injection.test.ts`
    - Test sort parameter validation rejects injection attempts
    - Test parameterized queries are used ($ placeholders)
    - Test known SQL injection payloads are blocked
    - Include edge cases (encoded attacks, comment syntax)
    - _Requirements: Security testing framework, SQL injection prevention_
  
  - [ ] 3.2 Create XSS prevention tests
    - Create `tests/security/xss-prevention.test.ts`
    - Test HTML escaping in user-generated content
    - Test script tag removal and attribute filtering
    - Test event handler stripping (onload, onerror, etc)
    - Include various XSS payload variants
    - _Requirements: Security testing framework, XSS prevention_
  
  - [ ] 3.3 Create rate limiting enforcement tests
    - Create `tests/security/rate-limiting.test.ts`
    - Test endpoint rate limits are enforced after threshold
    - Test rate limit headers are included in responses
    - Test per-user/IP isolation of limits
    - Test cleanup/reset of rate limit counters
    - _Requirements: Security testing framework, rate limiting enforcement_
  
  - [ ] 3.4 Create authentication security tests
    - Create `tests/security/auth-security.test.ts`
    - Test JWT token validation and expiry
    - Test refresh token rotation
    - Test sensitive headers not exposed in errors
    - Test CORS preflight handling
    - _Requirements: Security testing framework, authentication security_
  
  - [ ] 3.5 Create OWASP Top 10 compliance test suite
    - Create `tests/security/owasp-compliance.test.ts`
    - Test broken access control (role-based access)
    - Test cryptographic failures (password hashing, TLS)
    - Test insecure design patterns
    - Test security misconfiguration (headers, CORS)
    - _Requirements: Security testing framework, OWASP compliance_

- [ ] 4. Enhance GitHub Actions CI/CD pipeline
  - [ ] 4.1 Create enhanced backend CI workflow with coverage reporting
    - Update `.github/workflows/backend-ci.yml` to add coverage job
    - Add coverage collection with Jest coverage reports
    - Generate HTML coverage report artifact
    - Calculate coverage metrics (statements, branches, functions, lines)
    - _Requirements: GitHub Actions workflow structure, coverage reporting strategy_
  
  - [ ] 4.2 Add coverage threshold enforcement to workflow
    - Add step that validates 70% minimum coverage threshold
    - Add coverage delta calculation (compare to main branch)
    - Add step that fails workflow if threshold not met
    - Generate PR comment showing coverage impact
    - _Requirements: Coverage reporting strategy, CI enforcement_
  
  - [ ] 4.3 Add security scanning jobs to CI workflow
    - Add OWASP/security test job to workflow
    - Add SQL injection prevention test job
    - Add XSS prevention test job
    - Add rate limiting test job
    - Fail workflow on security findings (critical/high severity)
    - _Requirements: Security testing framework, CI/CD pipeline integration_
  
  - [ ] 4.4 Create test matrix configuration for parallel execution
    - Add Jest configuration for parallel test execution
    - Configure test splitting by directory (unit, integration, e2e)
    - Set up matrix strategy for different test types
    - Optimize execution time with caching
    - _Requirements: GitHub Actions workflow structure, matrix strategy_
  
  - [ ] 4.5 Add load test job to staging environment
    - Create new workflow job for k6 load testing
    - Add k6 installation and test execution steps
    - Configure load test to run against staging environment
    - Add performance baseline comparison logic
    - _Requirements: Load testing framework, CI/CD integration_

- [ ] 5. Create comprehensive E2E test suite for critical flows
  - [ ] 5.1 Create E2E test framework and helpers
    - Create `tests/e2e/helpers.ts` with common E2E utilities
    - Implement test user creation and authentication
    - Implement test data cleanup helpers
    - Add retry logic for eventual consistency
    - _Requirements: Critical E2E flows, E2E test helpers_
  
  - [ ] 5.2 Implement market creation to resolution E2E test
    - Create `tests/e2e/critical-flows/market-creation.e2e.test.ts`
    - Test market creation via API
    - Verify indexer ingests Stellar event
    - Verify database state after indexing
    - Test market status transitions
    - _Requirements: Critical E2E flows, market creation_
  
  - [ ] 5.3 Implement user authentication flow E2E test
    - Create `tests/e2e/critical-flows/auth-flow.e2e.test.ts`
    - Test user registration with email verification
    - Test login and JWT issuance
    - Test token refresh flow
    - Test logout and token revocation
    - _Requirements: Critical E2E flows, authentication flow_
  
  - [ ] 5.4 Implement bet placement and oracle resolution E2E test
    - Create `tests/e2e/critical-flows/bet-placement.e2e.test.ts`
    - Test placing bets on open market
    - Test odds calculation and pool updates
    - Test oracle resolution
    - Test settlement calculation
    - _Requirements: Critical E2E flows, bet placement_
  
  - [ ] 5.5 Implement winnings claim flow E2E test
    - Create `tests/e2e/critical-flows/winnings-claim.e2e.test.ts`
    - Test winnings calculation
    - Test claim request processing
    - Test payout verification
    - Test repeated claims are blocked
    - _Requirements: Critical E2E flows, winnings claim_
  
  - [ ] 5.6 Implement dispute resolution flow E2E test
    - Create `tests/e2e/critical-flows/dispute-resolution.e2e.test.ts`
    - Test dispute filing
    - Test admin dispute review
    - Test dispute resolution and settlement
    - Test event broadcasts to users
    - _Requirements: Critical E2E flows, dispute resolution_

- [ ] 6. Establish load testing framework and baselines
  - [ ] 6.1 Create load test baseline configuration and utilities
    - Create `tests/load/baselines/performance-baseline.json`
    - Implement `tests/load/utils/baseline-recorder.js`
    - Record baseline metrics (p50, p95, p99, throughput, errors)
    - Add baseline comparison logic
    - _Requirements: Load testing framework, baselines_
  
  - [ ] 6.2 Create WebSocket performance load test
    - Create `tests/load/websocket-perf.loadtest.js`
    - Configure 100 VUs, 2-minute duration
    - Test concurrent WebSocket connections
    - Measure connection time, message latency
    - Add thresholds (p95 < 500ms)
    - _Requirements: Load testing framework, WebSocket testing_
  
  - [ ] 6.3 Create activity feed load test
    - Create `tests/load/activity-feed.loadtest.js`
    - Configure 50 VUs, 3-minute duration
    - Test high-frequency activity updates
    - Measure throughput and error rates
    - Compare against baseline
    - _Requirements: Load testing framework, activity feed testing_
  
  - [ ] 6.4 Create market list load test with pagination
    - Create `tests/load/market-list.loadtest.js`
    - Configure 75 VUs, 2-minute duration
    - Test pagination and filtering
    - Test sorting by various parameters
    - Record baseline metrics
    - _Requirements: Load testing framework, market list testing_
  
  - [ ] 6.5 Create rate limit verification load test
    - Create `tests/load/rate-limit.loadtest.js`
    - Configure 200 VUs, 1-minute duration
    - Verify rate limits are enforced
    - Measure 429 response rate
    - Verify X-RateLimit headers
    - _Requirements: Load testing framework, rate limit testing_

- [ ] 7. Checkpoint - Ensure all tests pass and coverage meets threshold
  - Ensure all unit tests pass (`npm test`)
  - Ensure all integration tests pass (`npm run test:integration`)
  - Ensure coverage meets 70% threshold
  - Verify no security findings (critical/high)
  - Ask the user if questions arise before proceeding.

- [ ] 8. Create comprehensive test documentation
  - [ ] 8.1 Create fixture usage guide
    - Document all available factories
    - Add examples for common test scenarios
    - Document edge case factories
    - Update `tests/fixtures/README.md`
    - _Requirements: Test documentation, fixture documentation_
  
  - [ ] 8.2 Create security testing documentation
    - Create `tests/security/SECURITY_TESTS.md`
    - Document each security test category
    - Map tests to OWASP Top 10
    - Document how to add new security tests
    - _Requirements: Test documentation, security testing_
  
  - [ ] 8.3 Create E2E testing documentation
    - Create `tests/e2e/E2E_GUIDE.md`
    - Document critical flow E2E tests
    - Document helper utilities
    - Add examples for adding new E2E tests
    - _Requirements: Test documentation, E2E testing_
  
  - [ ] 8.4 Create load testing documentation
    - Create `tests/load/LOAD_TESTING.md`
    - Document how to run load tests locally
    - Document baseline interpretation
    - Document performance regression detection
    - _Requirements: Test documentation, load testing_
  
  - [ ] 8.5 Update main README with test running instructions
    - Add test execution commands to backend README
    - Add coverage threshold requirement documentation
    - Document pre-deployment test checklist
    - Link to detailed test documentation
    - _Requirements: Test documentation_

- [ ] 9. Final checkpoint - Verify CI/CD integration
  - Run full test suite locally and verify all pass
  - Verify GitHub Actions workflow runs successfully
  - Verify coverage report is generated and accessible
  - Verify security tests run in CI workflow
  - Verify E2E tests run successfully
  - Verify load tests execute and record baselines
  - Ask the user if questions arise.

## Notes

### Execution Strategy

**Phase 1 (Foundations)**: Tasks 1-2
- Establish test fixture framework and database isolation
- Enable building on solid foundation for all subsequent tests
- Estimated: 1-2 days

**Phase 2 (Security)**: Task 3
- Implement comprehensive security test suite
- Covers all major OWASP vulnerabilities
- Estimated: 1-2 days

**Phase 3 (CI/CD Enhancement)**: Task 4
- Integrate enhanced testing into GitHub Actions
- Enable automated feedback on coverage and security
- Estimated: 1-2 days

**Phase 4 (E2E & Load Testing)**: Tasks 5-6
- Create critical flow E2E tests
- Establish performance baselines
- Estimated: 2-3 days

**Phase 5 (Documentation)**: Tasks 8-9
- Document all testing patterns and procedures
- Update workflows and checklists
- Estimated: 1 day

### Technical Considerations

**Database Isolation**
- Use transactions for integration tests to avoid cleanup
- Separate test database prevents data corruption
- Redis namespace isolation prevents cache collisions

**Test Performance**
- Unit tests should run in <1 second
- Integration tests should run in <5 seconds per test
- E2E tests should run in <10 seconds per critical flow
- Load tests run separately after main suite

**CI/CD Optimization**
- Parallel execution of test types
- Caching of node_modules for speed
- Matrix strategy for different environments
- Artifact storage for coverage reports

**Security Coverage**
- All OWASP Top 10 categories covered
- Security tests run on every commit
- CI fails on critical/high findings
- Baseline established for vulnerability trends

### Known Risks and Mitigations

**Risk**: Flaky tests in concurrent E2E execution
- **Mitigation**: Run E2E tests sequentially (--runInBand)
- **Mitigation**: Implement retry logic for eventual consistency

**Risk**: Performance regression detection sensitivity
- **Mitigation**: 10% threshold for baseline regression
- **Mitigation**: Allow manual override for known infrastructure changes

**Risk**: Coverage threshold too strict
- **Mitigation**: Start at 70%, increase to 80% in future
- **Mitigation**: Allow exemption for legacy code with gradual improvement

### Testing Philosophy

The implementation follows these principles:
1. **Speed**: Fast feedback loop (unit tests < 1 min)
2. **Reliability**: No flaky tests, deterministic outcomes
3. **Isolation**: Tests don't interfere with each other
4. **Clarity**: Clear test names describe what they test
5. **Coverage**: Focus on critical paths and edge cases
6. **Security**: Security testing on every commit
7. **Documentation**: Clear guides for adding new tests
