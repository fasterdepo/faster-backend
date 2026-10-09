# Testing & CI/CD Enhancement - Design Document

## Overview

This design enhances the BANKERCHANGER backend testing and CI/CD infrastructure to provide comprehensive coverage reporting, automated security testing, load testing baselines, and improved test isolation. The system uses GitHub Actions as the orchestrator with integrated tools for unit/integration/E2E tests, code coverage reporting with 70% threshold enforcement, OWASP security scanning, SQL injection/XSS prevention validation, rate limiting verification, and load testing on staging.

**Key Objectives:**
1. Enforce 70% code coverage threshold across all code paths
2. Automate security vulnerability scanning and prevention checks
3. Establish load testing baselines and regression detection
4. Improve test data management and fixture patterns
5. Ensure test isolation with proper cleanup between test runs
6. Provide comprehensive E2E test coverage for critical user flows

## Architecture

### CI/CD Pipeline Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                   GitHub Actions Workflow                   │
│                  (Triggered on push/PR)                      │
└─────────────────────────────────────────────────────────────┘
                              │
                    ┌─────────┴─────────┐
                    │                   │
            ┌───────▼────────┐  ┌──────▼──────────┐
            │  Unit Tests    │  │ Integration     │
            │  (Jest)        │  │ Tests (Jest)    │
            │                │  │                 │
            │ • Fast          │  │ • DB Required   │
            │ • Isolated      │  │ • Service-level │
            │ • Fixtures      │  │ • Fixture setup │
            └────────┬────────┘  └────────┬────────┘
                     │                    │
                     └────────┬───────────┘
                              │
                    ┌─────────▼─────────┐
                    │ Coverage Report   │
                    │ (70% threshold)   │
                    └─────────┬─────────┘
                              │
            ┌─────────────────┼─────────────────┐
            │                 │                 │
        ┌───▼────┐    ┌──────▼──────┐   ┌─────▼────┐
        │ Security│   │ E2E Tests    │   │ Lint/Build
        │ Scan    │   │ (Jest)       │   │
        │         │   │              │   │
        │• OWASP  │   │• Full flow   │   │
        │• SQL inj│   │• DB/Redis    │   │
        │• XSS    │   │• Cleanup     │   │
        └────┬────┘   └──────┬───────┘   └─────┬────┘
             │                │                 │
             └────────────────┼─────────────────┘
                              │
                    ┌─────────▼─────────┐
                    │ Load Test         │
                    │ (k6 on staging)   │
                    │ • WebSocket perf  │
                    │ • Rate limiting   │
                    │ • Baseline record │
                    └───────────────────┘
```

### GitHub Actions Workflow Structure

**Primary Workflow: `backend-enhanced-ci.yml`**
- Triggered on: push to backend/**, PR to backend/**
- Concurrency: One per ref (cancel in-progress)
- Services: PostgreSQL 15, Redis 7
- Matrix: Node 20, test types (unit, integration, e2e, security)

**Key Workflow Jobs:**
1. `test-unit` - Unit tests with fast feedback
2. `test-integration` - Integration tests with DB/Redis
3. `coverage-report` - Coverage analysis and threshold enforcement
4. `security-scan` - OWASP/SQL injection/XSS validation
5. `test-e2e` - End-to-end critical flows
6. `load-test-staging` - Performance baseline on staging
7. `test-results` - Aggregated status

### Matrix Strategy

```yaml
test-matrix:
  test-type: [unit, integration, e2e, security]
  node-version: [20]
  db-version: [15]
```

## Components and Interfaces

### 1. Test Suite Organization

```
tests/
├── config/                    # Configuration tests
│   ├── cors.test.ts
│   └── env.test.ts
│
├── controllers/              # Controller unit tests
│   └── *.controller.test.ts
│
├── services/                 # Service unit tests
│   ├── *.service.test.ts
│   └── fixtures/
│       ├── service-fixtures.ts
│       └── mock-factories.ts
│
├── repositories/             # Repository unit tests
│   └── *.repository.test.ts
│
├── middleware/               # Middleware unit tests
│   └── *.middleware.test.ts
│
├── integration/              # Integration tests (require DB)
│   ├── *.integration.test.ts
│   └── setup.ts             # Shared DB setup
│
├── e2e/                      # End-to-end tests
│   ├── critical-flows/
│   │   ├── market-creation.e2e.test.ts
│   │   ├── bet-placement.e2e.test.ts
│   │   ├── oracle-resolution.e2e.test.ts
│   │   ├── winnings-claim.e2e.test.ts
│   │   └── dispute-resolution.e2e.test.ts
│   ├── setup.ts             # E2E test setup/cleanup
│   └── teardown.ts          # E2E cleanup scripts
│
├── security/                 # Security-focused tests
│   ├── sql-injection.test.ts
│   ├── xss-prevention.test.ts
│   ├── rate-limiting.test.ts
│   ├── auth-header-redaction.test.ts
│   ├── password-complexity.test.ts
│   └── owasp-compliance.test.ts
│
├── load/                     # Load testing scripts
│   ├── websocket-perf.loadtest.js
│   ├── activity-feed.loadtest.js
│   ├── market-list.loadtest.js
│   └── baselines/
│       └── performance-baseline.json
│
├── fixtures/                 # Shared test data factories
│   ├── user-factory.ts
│   ├── market-factory.ts
│   ├── bet-factory.ts
│   ├── trade-factory.ts
│   └── README.md            # Fixture documentation
│
└── utils/                    # Test utilities
    ├── db-cleanup.ts        # Transaction-based cleanup
    ├── test-db.ts           # Test database setup
    ├── mock-stellar.ts      # Stellar API mocking
    └── helpers.ts           # Common test helpers
```

### 2. Test Data Factory Pattern

**Factory Interface:**
```typescript
interface TestDataFactory<T> {
  one(overrides?: Partial<T>): T;
  batch(count: number, overrides?: Partial<T>): T[];
  invalid(): InvalidTestData<T>;
  edge(): EdgeCaseTestData<T>;
}
```

**Example - User Factory:**
```typescript
export const userFactory: TestDataFactory<User> = {
  one: (overrides) => ({ 
    id: randomUUID(),
    username: faker.person.firstName(),
    email: faker.internet.email(),
    password: 'SecurePass123!',
    ...overrides 
  }),
  batch: (count, overrides) => 
    Array.from({ length: count }, () => userFactory.one(overrides)),
  invalid: () => ({
    shortPassword: 'short',
    invalidEmail: 'not-an-email',
    missingUsername: { username: '' }
  }),
  edge: () => ({
    maxLengthUsername: 'a'.repeat(255),
    unicode: { username: '测试用户' },
    sql_injection: { username: "'; DROP TABLE users; --" }
  })
};
```

### 3. Test Isolation Strategy

**Database Transaction Wrapper:**
```typescript
describe('Isolated Integration Test', () => {
  let transaction: Transaction;

  beforeEach(async () => {
    transaction = await db.transaction();
  });

  afterEach(async () => {
    await transaction.rollback();  // No cleanup needed
  });

  it('should not affect other tests', async () => {
    // Test runs in isolated transaction
    // Automatic rollback ensures no state leakage
  });
});
```

**Test Database Setup:**
- Separate test database: `bankerchanger_test`
- Migration run before test suite
- Transaction-based isolation (no manual cleanup)
- Redis test instance with flushdb between suites

### 4. Coverage Reporting Strategy

**Coverage Thresholds (70% minimum):**
- Statements: 70%
- Branches: 70%
- Functions: 70%
- Lines: 70%

**Coverage Collection:**
- Jest coverage reports in JSON, HTML, and LCOV formats
- HTML reports stored as CI artifacts (accessible via GitHub Actions)
- LCOV reports sent to coverage services (Codecov integration)
- Coverage badges generated and embedded in README

**Enforcement:**
- CI workflow fails if coverage drops below 70%
- Coverage delta calculation (compare to main branch)
- Comments on PRs showing coverage impact
- Requires minimum coverage to merge

**Configuration (jest.config.json):**
```json
{
  "collectCoverageFrom": [
    "src/**/*.ts",
    "!src/**/*.d.ts",
    "!src/index.ts",
    "!src/types/**"
  ],
  "coverageThreshold": {
    "global": {
      "statements": 70,
      "branches": 70,
      "functions": 70,
      "lines": 70
    }
  }
}
```

### 5. Security Testing Framework

**Security Test Categories:**

#### A. SQL Injection Prevention
```typescript
describe('SQL Injection Prevention', () => {
  it('should reject SQL injection attempts in sort parameter', () => {
    const payload = { sort: "id' OR '1'='1" };
    const result = validateSortParameter(payload);
    expect(result.valid).toBe(false);
  });

  it('should safely parameterize database queries', async () => {
    const userInput = "test'; DROP TABLE bets; --";
    const query = buildUserQuery(userInput);
    expect(query.text).toContain('$1');  // Parameterized
    expect(query.values).toContain(userInput);
  });
});
```

#### B. XSS Prevention
```typescript
describe('XSS Prevention', () => {
  it('should escape HTML in user input', () => {
    const malicious = '<script>alert("xss")</script>';
    const sanitized = sanitizeUserInput(malicious);
    expect(sanitized).not.toContain('<script>');
  });

  it('should not render unescaped user content', () => {
    const xssPayload = '"><svg/onload=alert(1)>';
    const result = renderUserContent(xssPayload);
    expect(result).not.toContain('onload=');
  });
});
```

#### C. Rate Limiting Enforcement
```typescript
describe('Rate Limiting', () => {
  it('should block requests exceeding rate limit', async () => {
    const requests = Array(101).fill(null);  // 101 requests
    const responses = await Promise.all(
      requests.map(() => client.get('/api/markets'))
    );
    expect(responses.filter(r => r.status === 429).length).toBeGreaterThan(0);
  });

  it('should include rate limit headers', () => {
    const response = client.get('/api/markets');
    expect(response.headers['x-ratelimit-limit']).toBeDefined();
    expect(response.headers['x-ratelimit-remaining']).toBeDefined();
  });
});
```

#### D. Authentication Security
```typescript
describe('Auth Security', () => {
  it('should not expose sensitive auth headers in error responses', () => {
    const response = client.get('/protected', {
      headers: { Authorization: 'Bearer secret-token' }
    });
    expect(response.body).not.toContain('secret-token');
    expect(response.body).not.toContain('Bearer');
  });
});
```

#### E. OWASP Top 10 Compliance
- Broken Access Control: Role-based test coverage
- Cryptographic Failures: Password hashing, token validation
- Injection: SQL injection, command injection tests
- Insecure Design: Security by default tests
- Security Misconfiguration: Header validation tests

### 6. Load Testing Framework

**Tool: k6 (JavaScript/Go performance testing)**

**Load Test Structure:**
```javascript
// tests/load/websocket-perf.loadtest.js
import ws from 'k6/ws';
import { check, group, sleep } from 'k6';

export let options = {
  vus: 100,              // Virtual users
  duration: '2m',        // Test duration
  thresholds: {
    'ws_connecting': ['p(95)<500'],   // 95th percentile < 500ms
    'ws_session_duration': ['p(95)<5000'],
    'ws_errors': ['count<1'],         // No errors
  },
};

export default function () {
  group('WebSocket Connection', () => {
    const ws = new WebSocket('ws://staging.api/ws');
    ws.onopen = () => {
      check(ws, { 'ws connected': (w) => w.readyState === WebSocket.OPEN });
    };
    ws.close();
  });
}
```

**Baseline Configuration:**
- Baseline stored in `tests/load/baselines/performance-baseline.json`
- Each load test records: response times, error rates, throughput
- CI compares current results against baseline
- Regression alert if performance degrades >10%

**Load Test Scenarios:**
1. **WebSocket Perf**: 100 VUs, 2m duration, concurrent connections
2. **Activity Feed**: 50 VUs, 3m duration, high-frequency reads
3. **Market List**: 75 VUs, 2m duration, pagination/filtering
4. **Rate Limit**: 200 VUs, 1m duration, verify limits are enforced

### 7. Critical E2E Flows

**Critical Path Tests (Must Pass Before Deploy):**

1. **Market Creation to Resolution**
   - Create market via API
   - Verify indexer ingests Stellar event
   - Place bets as multiple users
   - Mock oracle resolution
   - Verify settlement calculation
   - Claim winnings

2. **User Authentication Flow**
   - Register user with email verification
   - Login and receive JWT
   - Refresh token after expiry
   - Logout and verify revocation

3. **Dispute Resolution Flow**
   - Create market and place bets
   - Trigger dispute via API
   - Admin reviews and resolves
   - Verify payout recalculation

4. **Admin Governance Actions**
   - Authenticate as admin
   - Perform admin actions (pause market, manual resolution)
   - Verify audit log entries
   - Verify WebSocket broadcasts to clients

## Data Models

### Coverage Report Model
```typescript
interface CoverageReport {
  timestamp: Date;
  statements: number;
  branches: number;
  functions: number;
  lines: number;
  threshold: number;
  passed: boolean;
  artifacts: {
    html: string;  // Path to HTML report
    lcov: string;  // Path to LCOV file
    json: string;  // Path to coverage JSON
  };
}
```

### Security Scan Result
```typescript
interface SecurityScanResult {
  scanType: 'sql-injection' | 'xss' | 'owasp' | 'rate-limit';
  testName: string;
  passed: boolean;
  findings: SecurityFinding[];
  timestamp: Date;
}

interface SecurityFinding {
  severity: 'critical' | 'high' | 'medium' | 'low';
  description: string;
  location: {
    file: string;
    line: number;
  };
  recommendation: string;
}
```

### Load Test Baseline
```typescript
interface PerformanceBaseline {
  scenario: string;
  metrics: {
    p50: number;    // 50th percentile latency (ms)
    p95: number;    // 95th percentile latency (ms)
    p99: number;    // 99th percentile latency (ms)
    throughput: number;  // requests/sec
    errorRate: number;   // percentage
  };
  timestamp: Date;
  version: string;
}
```

## Testing Strategy

### Test Distribution (Recommended)
- **Unit Tests**: 60% of test suite (~300 tests)
- **Integration Tests**: 30% of test suite (~150 tests)
- **E2E Tests**: 10% of test suite (~50 critical flows)
- **Security Tests**: Embedded throughout, ~50 dedicated tests

### Execution Strategy

**Fast Path (PR feedback, ~5 minutes)**
- Unit tests only (no DB)
- Lint and TypeScript check
- Security static analysis

**Full Suite (Merge validation, ~15 minutes)**
- Unit tests
- Integration tests
- Coverage enforcement
- Security tests
- E2E tests on main branch

**Extended Suite (Nightly, ~30 minutes)**
- All above
- Load tests (k6) on staging
- Performance regression detection
- Historical trend analysis

### Test Isolation and Cleanup

**Unit Tests (No Cleanup Required)**
- Pure functions, no side effects
- All dependencies mocked
- Execute in any order

**Integration Tests (Transaction-Based Cleanup)**
```typescript
describe('User Integration Tests', () => {
  let transaction: Transaction;

  beforeAll(async () => {
    // Setup test database if needed
  });

  beforeEach(async () => {
    transaction = await db.transaction();
  });

  afterEach(async () => {
    // Automatic rollback - no manual cleanup
    await transaction.rollback();
  });

  it('should create user', async () => {
    // Test runs in isolated transaction
  });
});
```

**E2E Tests (Explicit Cleanup Script)**
```typescript
afterAll(async () => {
  // Clean up created resources
  await cleanupMarkets(testDataIds.marketIds);
  await cleanupUsers(testDataIds.userIds);
  await db.close();
  await redis.disconnect();
});
```

**Cleanup Script (tests/e2e/teardown.ts)**
- Delete test data by ID prefix
- Clear Redis cache
- Close DB connections
- Verify no orphaned resources

### Fixture and Mocking Strategy

**Mocking Philosophy**
- Mock external services (Stellar, Boxing API, Email)
- Mock time for scheduled operations
- Keep database real for integration tests
- Use factories for consistent test data

**Mock Library: Jest**
- `jest.mock()` for module replacement
- `jest.fn()` for function spies
- `jest.spyOn()` for method interception

**Fixture Library: faker + custom factories**
- Generates realistic data
- Ensures consistency across tests
- Supports custom business rules

**Example Mock (Stellar Service)**
```typescript
jest.mock('../../src/services/StellarService', () => ({
  StellarService: jest.fn().mockImplementation(() => ({
    submitBet: jest.fn().mockResolvedValue({
      id: 'tx-12345',
      status: 'success'
    })
  }))
}));
```

## Error Handling

### Test Failure Classification

1. **Flaky Tests** (intermittent failures)
   - Identified by retry logic (3 attempts)
   - Flagged for investigation
   - Marked with `@flaky` tag

2. **Environmental Failures** (infrastructure)
   - Database connection failures
   - Redis unavailability
   - Auto-retry with exponential backoff

3. **Logical Failures** (code issues)
   - Assertion failures
   - Exceptions in test code
   - Requires code fix

### Error Reporting

**GitHub Check Annotations**
```
❌ Coverage Drop Detected
- Expected: 70%, Actual: 68.5%
- Critical files affected: BetService.ts (-2%)
- Action: Review coverage report artifact
```

**Coverage Report Artifacts**
- HTML report with line-by-line coverage
- LCOV format for CI integrations
- JSON format for programmatic analysis
- Accessible via GitHub Actions UI

## Documentation

### Fixture Documentation (tests/fixtures/README.md)
```markdown
# Test Fixtures Guide

## User Factory
```typescript
const user = userFactory.one({ role: 'admin' });
const users = userFactory.batch(5);
```

## Market Factory
```typescript
const market = marketFactory.one({ 
  status: 'open',
  outcomes: 2 
});
```

## Edge Cases
```typescript
const edge = userFactory.edge();  // Unicode, max length, etc.
const invalid = userFactory.invalid();  // Invalid data
```
```

### Test Coverage Documentation
- Coverage requirements for each module
- Coverage reporting dashboard link
- How to review coverage reports
- Coverage trend analysis

### Security Testing Documentation
- OWASP Top 10 mapping to test cases
- Security test checklist before deployment
- How to add new security tests
- Known vulnerabilities and mitigations

### Load Testing Documentation
- How to run load tests locally
- Understanding baseline comparisons
- Interpreting performance metrics
- Adding new load test scenarios

## Verification

### Pre-Deployment Checklist
- [ ] All tests passing (unit, integration, e2e)
- [ ] Coverage >= 70% threshold
- [ ] No security findings (critical/high)
- [ ] Load tests pass baseline regression
- [ ] No flaky test failures
- [ ] All fixtures cleaned up

### Monitoring and Alerts
- Coverage regression alert (drop > 5%)
- Security test failure alerts (critical/high)
- Load test regression alert (latency > 10%)
- Flaky test trend analysis (>2 failures/week)

## Notes

### Technology Stack
- **Test Framework**: Jest 29.7
- **Assertion Library**: Jest built-in
- **Mocking**: Jest mocks
- **Data Generation**: faker-js
- **Load Testing**: k6
- **Coverage**: Jest coverage
- **CI/CD**: GitHub Actions

### Future Enhancements
- Mutation testing for code quality
- Contract testing for API compatibility
- Visual regression testing for UI components
- Performance profiling on each commit
- Historical coverage trend dashboard
- Auto-generated test reports in PR comments
