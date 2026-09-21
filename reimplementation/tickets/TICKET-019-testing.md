# TICKET-019: Testing

## Metadata

| Field | Value |
|-------|-------|
| **Ticket ID** | TICKET-019 |
| **Title** | Testing |
| **Priority** | High |
| **Complexity** | XL (Extra Large) |
| **Dependencies** | TICKET-002, TICKET-003 (minimum); all tickets for full coverage |
| **Related User Stories** | (quality — supports all user stories) |

## Description

Set up the test infrastructure and create comprehensive test suites covering unit tests, integration tests, and end-to-end API tests for all modules. Testing should begin as soon as data models and authentication middleware are implemented (Phase 3) and grow incrementally as each ticket is completed.

## Technical Details

### Test Framework Configuration

**Runner:** `ts-mocha` (TypeScript-aware Mocha)
**Assertions:** `chai` (expect/should syntax)
**HTTP Testing:** `supertest` (Express endpoint testing)
**Mocking:** `sinon` (stubs, spies, mocks)
**HTTP Mocking:** `nock` (mock external HTTP requests)
**Reporter:** `mochawesome` (HTML test reports)

**Configuration file:** `.mocharc.yml`

```yaml
require:
  - ts-mocha
spec: src/tests/**/*.test.ts
timeout: 10000
reporter: mochawesome
reporter-options:
  reportDir: test-reports
  quiet: true
```

### Test Directory Structure

```
src/tests/
  setup.ts                          # Global test setup (DB connection, cleanup)
  fixtures/
    users.ts                        # Test user data
    participants.ts                 # Test participant data
    consents.ts                     # Test consent data
    privacyNotices.ts               # Test privacy notice data
    contracts.ts                    # Mock contract service responses
  unit/
    models/
      consent.test.ts               # Consent model validation, methods
      user.test.ts                  # User model, password hashing
      participant.test.ts           # Participant model
      userIdentifier.test.ts        # UserIdentifier model
      privacyNotice.test.ts         # PrivacyNotice model
    middleware/
      auth.test.ts                  # All 4 auth mechanisms
      joiValidation.test.ts         # Joi validation middleware
      errorHandler.test.ts          # Global error handler
    libs/
      crypto.test.ts                # Encryption/decryption round-trip
      jwt.test.ts                   # JWT generation/verification
      email.test.ts                 # Email service (mocked providers)
      jsonld.test.ts                # JSON-LD generation
    utils/
      checkUserIdentifier.test.ts   # Federation logic
      consentStateMachine.test.ts   # State machine transitions
  integration/
    users.test.ts                   # User API endpoints
    participants.test.ts            # Participant API endpoints
    consents.test.ts                # Consent API endpoints
    dataExchange.test.ts            # Data exchange endpoints
    privacyNotices.test.ts          # Privacy notice endpoints
    health.test.ts                  # Health check endpoint
```

### Test Setup (`setup.ts`)

```typescript
import mongoose from 'mongoose';
import { MongoMemoryServer } from 'mongodb-memory-server';

let mongoServer: MongoMemoryServer;

before(async () => {
  mongoServer = await MongoMemoryServer.create({ instance: { replSet: true } });
  await mongoose.connect(mongoServer.getUri());
});

afterEach(async () => {
  // Clean all collections between tests
  const collections = mongoose.connection.collections;
  for (const key in collections) {
    await collections[key].deleteMany({});
  }
});

after(async () => {
  await mongoose.disconnect();
  await mongoServer.stop();
});
```

### Unit Test Categories

#### Model Tests
- Schema validation (required fields, enum values, defaults)
- Pre-save hooks (password hashing on User)
- Instance methods (`validatePassword`, `isValid`, `toReceipt`)
- Index uniqueness (User email, Participant clientID)

#### Middleware Tests
- Each auth mechanism: valid token, expired token, missing token, invalid format
- Session shortcut paths
- Joi validation: valid body, missing fields, invalid types
- Error handler: each error class maps to correct HTTP status

#### Crypto Tests
- Encryption → decryption round-trip preserves data
- Different payloads produce different ciphertexts
- Invalid keys fail gracefully
- Signing → verification round-trip

#### Utility Tests
- `checkUserIdentifier`: user found, user not found, multiple users
- State machine: all valid transitions pass, all invalid transitions throw

### Integration Test Categories

#### User API Tests
- Signup flow (success, duplicate email, missing fields)
- Login flow (success, wrong password, non-existent user)
- Logout flow
- Profile retrieval
- Response field selection (no password leak)

#### Participant API Tests
- Registration (success, duplicate clientID, Joi validation failure)
- Login (success, invalid credentials)
- CRUD operations
- DSC integration (mocked external calls with nock)

#### Consent API Tests
- Privacy notice generation (mock Contract Service with nock)
- Consent grant via User Key
- Consent grant via User JWT
- Consent lifecycle: revoke, refuse, terminate, re-confirm, resume
- State machine enforcement (invalid transitions return 400)
- Consent querying with pagination
- Parent/child linking

#### Data Exchange Tests
- Full exchange flow with mocked DSC endpoints
- Token management (attach, validate)
- Error handling for failed external calls

### Test Fixtures

Create reusable test data factories:

```typescript
export function createTestUser(overrides?: Partial<IUser>): IUser {
  return {
    firstName: 'Test',
    lastName: 'User',
    email: `test-${Date.now()}@example.com`,
    password: 'TestPassword123!',
    ...overrides,
  };
}

export function createTestParticipant(overrides?: Partial<IParticipant>): IParticipant {
  return {
    legalName: 'Test Participant',
    clientID: `client-${Date.now()}`,
    clientSecret: 'TestSecret123!',
    ...overrides,
  };
}
```

### External Service Mocking

Use `nock` to mock all external HTTP calls:

```typescript
import nock from 'nock';

// Mock Contract Service
nock('http://contract-service:3001')
  .get('/contracts/123')
  .reply(200, mockContractResponse);

// Mock Provider DSC
nock('http://provider-dsc:8080')
  .post('/consent/export')
  .reply(200, { dataExchangeId: 'exchange-123' });
```

### Parameterized Tests

Use Mocha's array iteration for parameterized tests:

```typescript
const invalidTransitions = [
  { from: 'pending', to: 'revoked' },
  { from: 'refused', to: 'granted' },
  { from: 'revoked', to: 'granted' },
  { from: 'terminated', to: 'granted' },
];

invalidTransitions.forEach(({ from, to }) => {
  it(`should reject transition from ${from} to ${to}`, () => {
    expect(() => validateTransition(from, to)).to.throw(BadRequestError);
  });
});
```

### Files to Create

```
src/tests/
  setup.ts
  fixtures/
    users.ts
    participants.ts
    consents.ts
    privacyNotices.ts
    contracts.ts
  unit/
    models/consent.test.ts
    models/user.test.ts
    models/participant.test.ts
    models/userIdentifier.test.ts
    models/privacyNotice.test.ts
    middleware/auth.test.ts
    middleware/joiValidation.test.ts
    middleware/errorHandler.test.ts
    libs/crypto.test.ts
    libs/jwt.test.ts
    libs/email.test.ts
    libs/jsonld.test.ts
    utils/checkUserIdentifier.test.ts
    utils/consentStateMachine.test.ts
  integration/
    users.test.ts
    participants.test.ts
    consents.test.ts
    dataExchange.test.ts
    privacyNotices.test.ts
    health.test.ts
.mocharc.yml
```

### Dev Dependencies to Add

| Package | Purpose |
|---------|---------|
| `mongodb-memory-server` | In-memory MongoDB for tests |
| `nock` | HTTP request mocking |
| `sinon` | Stubs, spies, mocks |
| `@types/sinon` | Sinon type definitions |
| `@types/supertest` | Supertest type definitions |
| `mochawesome` | HTML test reporter |

## Acceptance Criteria

1. [ ] `pnpm run test` executes all test suites and produces a report
2. [ ] Test setup creates an in-memory MongoDB replica set
3. [ ] Database is cleaned between test cases (no test pollution)
4. [ ] Unit tests cover all 5 model schemas (validation, methods, hooks)
5. [ ] Unit tests cover all 4 authentication middleware paths
6. [ ] Unit tests cover encryption/decryption round-trip
7. [ ] Unit tests cover state machine transitions (valid and invalid)
8. [ ] Integration tests cover all API endpoints with supertest
9. [ ] External HTTP calls are mocked with nock (no real network calls in tests)
10. [ ] Parameterized tests are used for state machine transitions and validation
11. [ ] Test fixtures provide reusable factory functions
12. [ ] Test reports are generated in `test-reports/` directory
13. [ ] All tests pass with zero failures

## Known Issues to Address

1. **Test isolation**: The original test suite may have test pollution issues. Use `afterEach` cleanup to ensure test isolation.
2. **MongoDB replica set in tests**: `mongodb-memory-server` with `replSet: true` is required for Mongoose transaction support.
3. **Nock cleanup**: Always call `nock.cleanAll()` in `afterEach` to prevent cross-test interference.
4. **Timeouts**: Set appropriate test timeouts (10s for integration tests) to account for in-memory MongoDB startup.
