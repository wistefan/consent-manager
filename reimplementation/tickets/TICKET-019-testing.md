# TICKET-019: Testing

## Metadata

| Field | Value |
|-------|-------|
| **Ticket ID** | TICKET-019 |
| **Title** | Testing |
| **Priority** | High |
| **Complexity** | L (Large) |
| **Dependencies** | TICKET-002, TICKET-003 (minimum); all tickets for full coverage |
| **Related User Stories** | (quality — supports all user stories) |

## Description

Establish the test infrastructure and the suites that cover every module. Testing starts as soon as
the schema and identity layer exist and grows with each ticket.

Complexity drops from XL to L: two entire surfaces — data exchange and identity federation — left
the scope, and they carried the most combinatorial test burden in the pre-revision plan.

## Technical Details

### Stack

| Concern | Tool |
|---------|------|
| Framework | JUnit 5, with `junit-jupiter-params` |
| Micronaut integration | `micronaut-test-junit5` |
| Assertions | AssertJ |
| Database | Testcontainers PostgreSQL 16 |
| Identity provider | Testcontainers Keycloak |
| External HTTP | WireMock |
| Coverage | JaCoCo |

**Real PostgreSQL, not H2 or an in-memory substitute.** The schema relies on partial unique indexes,
`jsonb`, `timestamptz` and check constraints. A substitute that silently accepts a partial index it
does not enforce would let exactly the concurrency defects this design guards against pass the
suite.

**Real Keycloak, not hand-forged tokens.** The identity layer is the security boundary
(TICKET-003). Tests that mint their own tokens with a test key verify the test's assumptions rather
than the service's behaviour: they cannot exercise OIDC discovery, JWKS caching, key rotation, or
issuer rejection.

Both containers are started once per suite via a shared singleton, not per test class.

### Test Layers

```
src/test/java/eu/prometheusx/consentmanager/
  support/
    PostgresTestResource.java        # singleton container, schema reset between tests
    KeycloakTestResource.java        # singleton container, realm import
    TokenFactory.java                # obtains real tokens per role from Keycloak
    ContractServiceStubs.java        # WireMock stubs for contract fixtures
    TestDataFactory.java             # entity builders
  unit/
    ConsentStateMachineTest.java
    ConsentFactoryTest.java
    ClaimMapperTest.java
    CanonicalJsonSerializerTest.java
    ConsentSigningServiceTest.java
    ConsentReceiptServiceTest.java
    ContractContentExtractorTest.java
  integration/
    IdentityAndAccessIT.java
    UserManagementIT.java
    ParticipantManagementIT.java
    PrivacyNoticeIT.java
    ConsentGrantIT.java
    ConsentLifecycleIT.java
    ConsentQueryIT.java
    NotificationIT.java
    ObservabilityIT.java
  concurrency/
    ConcurrentRegistrationTest.java
    ConcurrentRegrantTest.java
```

### Parameterized Tests

Per project convention, anything exercising the same behaviour across multiple inputs is written as
`@ParameterizedTest`. The obligatory ones:

| Suite | Matrix |
|-------|--------|
| State machine | Every (from-state × to-state) pair — 7 × 7 — asserting exactly the nine valid transitions pass |
| Authorization | Every (endpoint × role × ownership) triple, asserting 200 / 401 / 403 / 404 |
| Token validation | Unknown issuer, bad signature, expired, not-yet-valid, wrong audience, `alg: none`, symmetric alg, missing role claim, missing identifier claim |
| Registration rule | All four entry states — unknown user, known unlinked, known linked, concurrent |
| Validation | Every constrained DTO field against valid, empty, oversized and malformed input |
| Receipts | Both formats × signing algorithms × tamper cases |
| Error mapping | Every exception type to its documented status |

### Concurrency Tests

The design leans on database constraints for correctness under concurrency, so those paths need
tests that actually run in parallel:

1. **Concurrent user registration** — N threads register the same identifier; exactly one user row
   results and every caller gets a success response.
2. **Concurrent re-grant** — two threads grant consent for the same (user, privacy notice); exactly
   one ends `GRANTED`, the other is rejected or supersedes cleanly, and no state has two active
   consents.
3. **Concurrent privacy notice generation** — two threads generate for the same contract and
   provider; exactly one active notice results.

Each asserts the final database state, not just the response codes.

### Security Tests

These are not optional extras. With the service out of the data path, authorization is the only
control it has:

1. A participant cannot read, terminate or query a consent it is not party to.
2. A participant cannot read a user it is not linked to, and gets 404 rather than 403 so it cannot
   probe for existence.
3. A user cannot read or act on another user's consent.
4. No endpoint accepts an identity supplied in the body, path or query in place of the token.
5. A token valid for one issuer is rejected when presented with another issuer's `kid`.
6. Signing-key rotation at the IDP is picked up without a restart.
7. A repeated unknown `kid` does not cause unbounded JWKS refetching.
8. Emitted logs contain no token, key material, or personal identifier above `DEBUG`.

### External Service Stubbing

All Contract Service interaction is stubbed with WireMock. **No test makes a real outbound network
call**, and no test-only route exists in the application (TICKET-015).

Stub fixtures cover: a well-formed bilateral contract, a well-formed ecosystem contract, a contract
with missing optional sections, an unresolvable service offering reference, a 500 response, and a
timeout.

### Test Isolation

Schema state is reset between test methods by truncating all tables in FK-safe order and restarting
sequences — faster and more reliable than recreating the schema, and it keeps the Flyway migration
verified once per suite rather than per test.

WireMock stubs are reset after every test. Keycloak state is shared read-only; tests that need a
distinct user or client create one with a unique name rather than mutating the imported realm.

### Coverage

JaCoCo with a build-failing threshold. Line coverage is a weak signal on its own, so the gate is
paired with named obligations: the state machine, claim mapping, authorization checks and the
signing path each require branch coverage, and a missing branch there fails the build regardless of
the overall percentage.

### Files to Create

As listed under Test Layers, plus:

```
src/test/resources/
  application-test.yml               # points at the Testcontainers instances
  contracts/*.json                   # contract fixtures
  logback-test.xml
```

## Acceptance Criteria

1. [ ] `./gradlew test` runs the full suite and produces a JaCoCo report
2. [ ] Tests run against Testcontainers PostgreSQL 16; no in-memory database substitute is used
3. [ ] Integration tests authenticate with real tokens from Testcontainers Keycloak
4. [ ] Containers start once per suite, not per test class
5. [ ] Schema state is reset between test methods with no cross-test pollution
6. [ ] No test makes a real outbound network call
7. [ ] Parameterized tests cover the full 7 × 7 state transition matrix
8. [ ] Parameterized tests cover the (endpoint × role × ownership) authorization matrix
9. [ ] Parameterized tests cover the token validation failure matrix
10. [ ] Concurrent registration of one identifier yields exactly one user row
11. [ ] Concurrent re-grant yields exactly one `GRANTED` consent
12. [ ] Concurrent notice generation yields exactly one active notice
13. [ ] All eight security tests pass
14. [ ] Signing-key rotation is exercised against a live IDP container
15. [ ] Flyway migrations are verified against an empty database once per suite
16. [ ] A log-scanning test finds no token, key material or personal identifier above `DEBUG`
17. [ ] Branch coverage obligations on the state machine, claim mapping, authorization and signing are met
18. [ ] The suite is deterministic: ten consecutive runs produce identical results

## Changes from the Pre-Revision Ticket

- Mocha/Chai/Supertest/Sinon/nock replaced by JUnit 5 / Micronaut Test / AssertJ / WireMock.
- `mongodb-memory-server` replaced by Testcontainers PostgreSQL — a real database, because the
  design depends on real constraints.
- Keycloak added: real tokens rather than hand-forged ones.
- Concurrency tests added, since correctness under concurrency is now enforced by database
  constraints rather than application checks.
- Security tests made an explicit, enumerated obligation.
- Data exchange and federation suites removed with their features.
- Parameterized testing is the default form, per project convention, not an illustrative example.
