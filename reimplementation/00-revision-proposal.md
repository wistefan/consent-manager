# Revision Proposal: Scope and Technology Realignment

**Status:** Proposal — awaiting decisions on the open questions in Section 2
**Date:** 2026-09-29
**Affects:** all documents in `reimplementation/`, all 64 user stories, all 19 tickets

This document analyses the existing reimplementation plan (documents 01–08 and the 19 tickets)
against the revised requirements and proposes concrete updates. It is a change proposal, not a
replacement — once the decisions in Section 2 are made, the underlying documents are rewritten
accordingly and this file becomes the record of *why*.

---

## 1. Impact Summary

The revised requirements are not incremental. Three of them are structural:

| Change | Consequence |
|--------|-------------|
| **TypeScript/Express/MongoDB → Java/Micronaut/PostgreSQL** | Documents 01, 02 and tickets 001, 002, 015, 018, 019 are rewritten end to end. ~18 of the 40 catalogued "known bugs" are Mongoose/Express artefacts and disappear by construction. |
| **Authentication moves to an external OIDC provider** | The four in-house auth mechanisms (participant JWT, user JWT, user key header, catalog shared secret), password handling, sessions, and the OAuth module all disappear. Replaced by one mechanism: validate a bearer token against a registered IDP. Document 04 and tickets 003, 004, 016 are rewritten. |
| **Data exchange leaves the scope** | Document 06 is deleted, ticket 012 is deleted, 6 stories are dropped, and 4 further stories in the consent module that are actually data-exchange protocol are dropped with them. |

Plus two model-level changes:

- **`UserIdentifier` as an entity disappears.** A user is one record with one globally unique
  identifier and a set of participants it is known to. The federation/reconciliation machinery
  (`checkUserIdentifier`, `TryFindMatchingUser`, the four email-reconciliation cases, implicit
  federation in middleware) goes away entirely — roughly 600 lines of documented behaviour and
  8 of the 13 user-management stories.
- **Participants carry a data space identifier and nothing data-space-operational.** No
  clientID/secret, no DSC endpoints, no self-description push, no DSC login.

Net story count: **64 → 53** (21 removed, ~20 substantially rewritten, 10 new).
Net ticket count: **19 → 15** (5 deleted, 1 added, 9 rewritten, 4 revised).

---

## 2. Decisions (Settled 2026-09-29)

These were open when the proposal was drafted; all are now decided. Recorded here because they
shape the rest of the document.

| # | Question | Decision |
|---|----------|----------|
| **D1** | IDP registry: static configuration or a runtime admin API? | **Static configuration only.** Trusted issuers, discovery URLs, claim mappings and role mappings live in `application.yml`; changing the trust list is a redeploy. No `idp_registrations` table, no bootstrap-admin problem. `US-ID-009` is therefore dropped. |
| **D2** | Participant onboarding: self-service or admin-gated? | **Self-service from the token.** Any token carrying the `PARTICIPANT` role may create and update its own record. The participant identifier is always taken from the token, never from the request body. No `ADMIN` role, no activation workflow. |
| **D3** | Persistence: Micronaut Data JDBC or JPA/Hibernate? | **Micronaut Data JDBC + Flyway.** The object graph is shallow, compile-time repositories avoid lazy-loading surprises, and JSONB covers the contract-derived payloads. |
| **D4** | `contract-agent` is a Node package and cannot run on the JVM. | **Dropped from scope.** `US-SM-001`, `US-SM-002`, `US-SM-003` and `TICKET-014` are removed. Nothing else in the plan depends on them; recommendations are a separate product concern. |
| **D5** | Should the user↔participant link carry the participant's own internal ID? | **Yes** — optional `local_identifier` column, stored as a convenience, never used as a lookup key by any other participant. |
| **D6** | Greenfield, or replacing a live instance with data? | **Greenfield.** No migration ticket. Identifier collisions are an onboarding-policy question for the data space, not a migration concern — see Section 10.2. |
| **D7** | Is the OpenAPI document generated from the code, or is the code generated from the document? | **API-first.** `api/openapi.yaml` is authored and reviewed as the design artefact; API interfaces and request/response models are generated from it. Controllers implement the generated interfaces, so spec and code cannot diverge without a compile error. The `micronaut-openapi` annotation processor is not used. |

---

## 3. Target Technology Stack

Replaces Section 3 of `01-architecture-overview.md`.

| Component | Current | Target |
|-----------|---------|--------|
| Language | TypeScript | **Java 21 (LTS)** |
| Framework | Express 4.18 | **Micronaut 4.x** |
| Build | pnpm + tsc | **Maven** — a declarative POM, so the build carries no scripting language; Java is the only language in the repository |
| Database | MongoDB 7 (replica set) | **PostgreSQL 16** |
| Data access | Mongoose 7.3 | **Micronaut Data JDBC** |
| Schema management | implicit (schemaless) | **Flyway** versioned migrations |
| Authentication | JWT + bcrypt + express-session | **micronaut-security-jwt**, remote JWKS, OIDC discovery |
| Sessions | connect-mongo | **none** — stateless resource server |
| Validation | Joi middleware | **Jakarta Bean Validation** on DTOs (`@Valid`) |
| HTTP client | Axios + cache interceptor | **Micronaut declarative `@Client`** + Caffeine cache |
| API contract | hand-maintained `swagger.json`, unenforced | **API-first**: `api/openapi.yaml` authored, Java interfaces and models generated from it (OpenAPI Generator, `java-micronaut-server`) |
| Logging | Winston + daily-rotate-file | **SLF4J/Logback**, JSON to stdout (12-factor) |
| Metrics/health | custom `/health` | **micronaut-management** + Micrometer/Prometheus |
| Email | Mandrill + Nodemailer | **Micronaut Email** (SMTP via Jakarta Mail; provider module optional) |
| Templating | EJS | **Micronaut Views** (Velocity/Thymeleaf) — only if HTML receipts are kept |
| Testing | Mocha/Chai/Supertest/Sinon/nock | **JUnit 5 (parameterized) + Micronaut Test + Testcontainers + WireMock** |
| Packaging | Docker (node) | **Docker (distroless JRE)**, optional GraalVM native image |

Notes:

- Daily-rotate log files are dropped deliberately: in a containerised deployment, log rotation is
  the platform's job and structured stdout is what the platform can index.
- The API is **specification-first**: the OpenAPI document is authored and reviewed as the design
  artefact, and the API interfaces and request/response models are generated from it. Controllers
  implement the generated interfaces, so an operation added or changed in the spec is a compile
  error until the code matches. Validation constraints and the RFC 7807 error contract are declared
  once, in the schema. The `micronaut-openapi` annotation processor is deliberately **not** used —
  deriving the contract from the code would make it an afterthought and allow implementation detail
  to leak into what integrators build against. Ticket 015 changes meaning accordingly.
- Testcontainers gives real PostgreSQL in tests; a Keycloak container covers the IDP integration
  tests. This removes the current need for a MongoDB replica set in CI.
- Per project convention, tests are written as JUnit 5 `@ParameterizedTest` wherever the same
  behaviour is exercised over multiple inputs (state transitions, claim validation, role mapping).

---

## 4. Identity and Access Architecture (new)

This replaces document `06-data-exchange.md`, reusing its slot as
**`06-authentication-and-identity.md`**.

### 4.1 Principles

1. The Consent Manager is an **OAuth2 resource server only**. It issues no tokens, stores no
   credentials, and has no login endpoints.
2. Trust is anchored in a **registered IDP**, identified by its `issuer` and configured through its
   `.well-known/openid-configuration` endpoint. The registry is **static configuration** (D1).
   Signing keys are resolved from the discovered JWKS URI and cached with rotation support.
3. A **user is identified by one globally unique string**. It may be an email address, a DID, a URI,
   or a hash derived from an EUDI Wallet PID. The Consent Manager treats it as opaque.
4. There is **no signup**. A user record exists because a participant registered the identifier, or
   because a token bearing that identifier was presented (just-in-time provisioning).
5. Authorization is **endpoint-level and role-based** today, structured so that RBAC/ABAC on
   resource attributes can be layered on later without changing the token contract.

### 4.2 Token Contract — User Access Token

Documented as a first-class deliverable; integrators need it to configure their IDP.

| Claim | Required | Purpose |
|-------|----------|---------|
| `iss` | yes | Must exactly match a registered IDP issuer. Selects the JWKS and the claim mapping. |
| `aud` | yes | Must contain the configured Consent Manager audience. |
| `exp`, `iat` | yes | Standard lifetime validation with configurable clock leeway. |
| `sub` | yes | Subject at the IDP. |
| *user-identifier claim* | yes | The globally unique user identifier. Claim name is configurable per IDP; defaults to `sub`. |
| *roles claim* | yes | Must contain the role mapped to `USER`. Claim path configurable (e.g. `realm_access.roles`). |
| `email` | no | Stored for optional search and notification. Never used as primary key. |
| `email_verified` | no | Recorded; does not gate consent. |
| `name`, `given_name`, `family_name` | no | Display only. |
| `nbf`, `jti` | no | Validated if present. |

### 4.3 Token Contract — Participant Access Token

| Claim | Required | Purpose |
|-------|----------|---------|
| `iss`, `aud`, `exp`, `iat`, `sub` | yes | As above. |
| *participant-identifier claim* | yes | The data space global participant ID (DID, URI, or IDP-issued string). Configurable; defaults to `sub`. |
| *roles claim* | yes | Must contain the role mapped to `PARTICIPANT`. |
| `azp` / `client_id` | no | Recorded for audit. |

A third role, `CATALOG`, replaces the `x-catalog-consent-key` shared secret for catalog-initiated
calls (user search, consent resume).

### 4.4 Configuration Sketch

```yaml
consent-manager:
  identity-providers:
    - issuer: "https://idp.example.org/realms/dataspace"
      discovery-url: "https://idp.example.org/realms/dataspace/.well-known/openid-configuration"
      audience: "consent-manager"
      jwks-cache-ttl: 1h
      claims:
        user-identifier: "sub"
        participant-identifier: "participant_id"
        roles: "realm_access.roles"
      role-mapping:
        USER: "dataspace-user"
        PARTICIPANT: "dataspace-participant"
        CATALOG: "dataspace-catalog"
```

### 4.5 What This Removes

| Removed | Replaced by |
|---------|-------------|
| `verifyParticipantJWT` (incl. `serviceKey`/`clientSecret` path) | IDP token with `PARTICIPANT` role |
| `verifyUserJWT` (session / `x-user-key` / OAuth paths) | IDP token with `USER` role |
| `verifyUserKey` (`x-user-key` header) | Nothing — a participant can no longer act as a user |
| `consentKeyCheck` (`x-catalog-consent-key`) | IDP token with `CATALOG` role |
| `express-session` + MongoDB session store | Nothing — stateless |
| bcrypt password hashing, `SALT_ROUNDS` | Nothing |
| OAuth authorize/token endpoints (dead code) | Nothing — the IDP is the authorization server |
| `?participant=<token>` query-parameter auth | `Authorization: Bearer` only |

---

## 5. Domain Model Changes

### 5.1 Entities

| Current | Target |
|---------|--------|
| `User` (email PK-ish, password, firstName, lastName, identifiers[], jsonld) | `users` — `identifier` (globally unique, natural key), optional `email`, optional name fields. No password, no JSON-LD. |
| `UserIdentifier` (email, identifier, attachedParticipant, user) | **Deleted.** Replaced by `user_participants` (user_id, participant_id, optional `local_identifier`). |
| `Participant` (clientID, clientSecret, dataspaceEndpoint, endpoints{4}) | `participants` — `identifier` (global, natural key), `legal_name`, `self_description_uri`, `email`, optional contact/notification endpoint. No secrets, no DSC endpoints. |
| `Consent` (30+ fields incl. token, dataExchangeId) | `consents` + `consent_events`. Data-exchange fields removed. |
| `PrivacyNotice` | `privacy_notices` — relational columns for what is queried, JSONB for the contract-derived payload. |

### 5.2 Schema Sketch

```
users(id uuid pk, identifier text unique not null, email text, first_name text,
      last_name text, created_at, updated_at)

user_participants(user_id uuid fk, participant_id uuid fk, local_identifier text,
      created_at, primary key (user_id, participant_id))

participants(id uuid pk, identifier text unique not null, legal_name text not null,
      self_description_uri text, email text, endpoints jsonb, created_at, updated_at)

privacy_notices(id uuid pk, contract_uri text not null, provider_id uuid fk,
      consumer_id uuid fk, payload jsonb not null, archived_at timestamptz,
      created_at, updated_at)
      -- unique partial index on (contract_uri) where archived_at is null

consents(id uuid pk, user_id uuid fk, privacy_notice_id uuid fk, provider_id uuid fk,
      consumer_id uuid fk, status text not null check (...), consented boolean not null,
      parent_consent_id uuid fk null, created_at, updated_at)

consent_events(id uuid pk, consent_id uuid fk, event_type text not null,
      actor text, occurred_at timestamptz not null, details jsonb)
```

Design notes:

- **Enums as `text` + `CHECK`**, not PostgreSQL `ENUM` types — adding a state should be a one-line
  migration, not a type alteration.
- **Consent events become a table**, not an embedded array. They are the audit trail; they need to
  be queryable and append-only.
- **Primary keys are UUIDs**, but `users.identifier` and `participants.identifier` are the natural
  keys and carry unique constraints. External APIs address entities by the global identifier;
  UUIDs stay internal.
- No `schema_version` field (Flyway owns schema versioning).

### 5.3 Behavioural Rule Replacing Federation

> When a participant registers user identifier `X`: if no user with `identifier = X` exists, create
> one and link it to the participant. If one exists, add the participant to its links. Never merge,
> never match on email, never guess.

This single rule replaces `checkUserIdentifier`, `TryFindMatchingUser`, the four email
reconciliation cases, implicit federation in `verifyUserJWT`, and `US-UM-013`. It is deterministic
and idempotent — a considerable reduction in both code and failure modes.

---

## 6. Per-Module Changes

### 6.1 Consent Management (doc 03)

| Section | Action |
|---------|--------|
| 1. Lifecycle state machine | **Keep.** Port as an explicit, table-driven transition guard. |
| 2. Privacy notice generation | **Keep.** Still contract-driven; port the contract-service client. |
| 3. Grant flow (user key) + 4. Grant flow (user JWT) | **Merge into one flow.** See Section 7, US-CM-003. |
| 5. Email verification flow (4 reconciliation cases) | **Delete.** Email verification is the IDP's responsibility. |
| 6–10. Revoke / refuse / terminate / re-confirm / resume | **Keep**, with auth reworked to roles. |
| 11. Consent receipt formats | **Keep.** ISO/IEC 29184 + Kantara. |
| 12. Consent querying | **Keep**, keyed by global user identifier. |
| 13. PDI iframe redirect | **Demote to Could-have** or drop — it is a frontend concern and the service has no frontend. |
| 14. Available exchanges | **Keep, rename** to "Available Privacy Notices / Contract Discovery". The current name implies data exchange; the behaviour is contract lookup. |
| 15. Token management (attach/validate data exchange token) | **Delete** — data exchange protocol. |
| 16. Consent encryption (AES-256-CBC + RSA to DSCs) | **Replace.** Payload encryption existed to protect transport to DSCs. With data exchange gone, what remains of value is **receipt signing**: sign consent receipts as JWS and publish the verification key at `/.well-known/jwks.json` instead of a bespoke PEM endpoint. |
| 17. Endpoint reference | Regenerate. |
| 18. Known bugs | Re-triage (Section 9). |

**New capability worth adding:** a consent **revocation notification** to the provider and consumer
participants (webhook or outbox). Revocation that nobody is told about is not effective revocation.
This is a consent concern, not data exchange, and it is the one useful thing the removed DSC
integration was doing. Proposed as `US-CM-025`.

### 6.2 User Management (doc 04)

Rewritten. Sections 1–3 (signup/login/logout), 7–13 (UserIdentifier CRUD + federation), 15 (JSON-LD)
are removed or collapsed. What remains: JIT provisioning, registration (single/bulk), search,
profile, participant links.

### 6.3 Participant Management (doc 05)

| Section | Action |
|---------|--------|
| 1. Registration | **Rewrite** — identifier from token, no clientID/secret, no DSC bootstrap. |
| 2. Login | **Delete** — replaced by IDP documentation. |
| 3. Retrieval | **Keep**, lookup by global identifier instead of clientID. |
| 4. Update | **Keep**, endpoint set reduced. |
| 5. Deletion | **Keep**, and fix the documented cascade gap (FK `ON DELETE` policy is now explicit in the schema — a genuine benefit of the move to PostgreSQL). |
| 6. Public key management | **Rewrite** as JWKS. |
| 7. DSC integration | **Delete.** |
| 8. Joi validation | **Rewrite** as Bean Validation. |
| 9. `verifyParticipantJWT` | **Delete**, see Section 4. |
| 10–11. Field selection / endpoints | Regenerate. |
| 12. Known bugs | 5 of 7 vanish with the design change. |

### 6.4 Data Exchange (doc 06)

**Deleted.** The document is removed and the slot reused for
`06-authentication-and-identity.md`. Rationale, recorded in the README: the actual transfer of data
is executed between participants; the Consent Manager's role ends at producing a verifiable,
queryable consent record that participants can check before and during an exchange.

The one thing worth preserving from it is the *interface* by which a participant asks "is there a
valid consent for user U, contract C, right now?" — that is already covered by consent querying
(US-CM-013), and should be called out explicitly as the replacement contract.

### 6.5 Supporting Modules (doc 07)

| Section | Action |
|---------|--------|
| 1. OAuth module | **Delete.** |
| 2. Consent Agent | **Delete** (D4) — `contract-agent` has no JVM path. |
| 3. Email service | **Shrink.** Only pending-consent notification and (optionally) revocation notices remain. Mandrill-specific behaviour dropped; SMTP by default. |
| 4. Session management | **Delete.** |
| 5. Logging | **Rewrite** — SLF4J/Logback, JSON to stdout, correlation/trace IDs. |
| 6. Request validation | **Rewrite** as Bean Validation; demoted from a story to an acceptance criterion (see US-SM-007 below). |
| 7. Error handling | **Keep** — Micronaut `@Error` handlers, RFC 7807 Problem Details recommended over the ad-hoc shape. |
| 8. JSON-LD self-descriptions | **Drop for users** (there is no signup to generate one at, and identity is external). **Keep for participants** as an opaque stored URI. |
| 9. Simulated contract service | **Keep** as a test fixture — but move it to WireMock stubs in the test source set rather than a route mounted in the production app. The current arrangement is the cause of documented bug 13.1. |
| 10. Swagger | **Rewrite** — spec authored first, code generated from it. |
| 11. Health check | **Rewrite** — `micronaut-management`, with a readiness probe that includes the PostgreSQL connection and IDP discovery reachability. |
| 12. Server setup | Rewrite. |

---

## 7. Story-by-Story Disposition

Legend: **Keep** (port as-is) · **Revise** (same intent, changed criteria) · **Rewrite** (same ID,
new substance) · **Remove** · **Merge**

### 7.1 Consent Management (24 → 20)

| ID | Title | Action | Note |
|----|-------|--------|------|
| US-CM-001 | Generate privacy notice between parties | Revise | Parties addressed by global participant identifier; drop base64-encoded self-description URLs from the path in favour of plain URI query parameters. |
| US-CM-002 | Retrieve privacy notice populated | Keep | |
| US-CM-003 | Grant consent via user key | **Rewrite + Merge** | Becomes **"Grant consent as the authenticated data subject"**, absorbing US-CM-004. Keyed by the global user identifier from the token, not by email, and not by a participant-supplied header. |
| US-CM-004 | Grant consent via user JWT | **Repurpose** | Merged into 003. The ID is reused for **"Participant requests consent from a user"** — creates a consent in `pending` and notifies the user. This preserves the participant-initiated use case without letting a participant grant consent on a user's behalf, which the old user-key flow effectively permitted. |
| US-CM-005 | Verify consent via email | **Remove** | Identity verification belongs to the IDP. |
| US-CM-006 | Revoke consent | Keep | |
| US-CM-007 | Refuse consent | Keep | |
| US-CM-008 | Terminate consent | Keep | |
| US-CM-009 | Re-confirm consent | Revise | Drop the `triggerDataExchange` parameter. |
| US-CM-010 | Resume pending/draft consent | Revise | Auth via `CATALOG` role; drop `dataExchangeId`. |
| US-CM-011 | List my consents | Revise | Subject resolved from token. |
| US-CM-012 | Get consent by ID | Keep | |
| US-CM-013 | Query consents as participant | Revise | User addressed by global identifier; participant scope from token. **Promote** — this is now the primary integration point for participants replacing the data-exchange flow. |
| US-CM-014 | Redirect to PDI iframe | Demote / Remove | Could-have at best. |
| US-CM-015 | Discover available exchanges as user | Revise + rename | "Discover available contracts/notices as user". |
| US-CM-016 | Discover available exchanges as participant | Revise + rename | Likewise. |
| US-CM-017 | Attach data exchange token | **Remove** | Data exchange protocol. Not in your list, but belongs with the dropped section. |
| US-CM-018 | Validate data exchange token | **Remove** | Same. |
| US-CM-019 | Generate ISO/IEC 29184 receipt | Keep | |
| US-CM-020 | Encrypt consent for transmission | **Rewrite** | Becomes **"Sign consent receipts (JWS) and publish the verification key"**. |
| US-CM-021 | Inherit parent consent on re-grant | Keep | |
| US-CM-022 | Generate privacy notice by contract URI | Revise | Plain URI parameter instead of base64. |
| US-CM-023 | Trigger inline data exchange on grant | **Remove** | As requested. |
| US-CM-024 | State machine enforcement | Keep | |
| **US-CM-025** | *(new)* Notify participants of consent state changes | **New** | Should-have. Webhook/outbox to provider and consumer on grant and revoke. |

### 7.2 User Management (13 → 10)

| ID | Title | Action | Note |
|----|-------|--------|------|
| US-UM-001 | User signup | **Rewrite** | **"Self-register as a data subject"** — no password, no credentials. A user record is provisioned just-in-time from the token's claims on first authenticated call, or explicitly via a self-registration endpoint that takes only the token. Name and email come from the token, are treated as non-authoritative display data, and are refreshed on each login. **Note the tension in the requirements:** "users can register themselves" (US-UM-001) vs "an explicit user signup should not exist" (User Management section). The proposed resolution is that self-registration is *implicit* — there is an endpoint, but it creates no credentials and can be called idempotently; a user who never calls it is still created on first use. |
| US-UM-002 | User login | **Rewrite** | **"Authenticate with an external IDP token"** — acceptance criteria cover issuer trust, signature via JWKS, audience, expiry, required claims, and the 401/403 distinction. |
| US-UM-003 | User logout | **Remove** | Stateless; token invalidation is the IDP's concern. Worth one documentation line pointing integrators at the IDP's end-session endpoint. |
| US-UM-004 | View my profile | Revise | No token refresh in the response (the IDP issues tokens). |
| US-UM-005 | Look up user by ID | Revise | By global identifier; **authentication now required** — the current story makes it a public endpoint, which leaks the existence of identifiers. |
| US-UM-006 | Search user by email | Revise | Demoted to Should-have/optional as requested; **US-UM-006b** (new, Must-have) covers search by global identifier. Suggest instead: one story, "search users by identifier (required) or email (optional)". |
| US-UM-007 | Register single user identifier | **Rewrite** | Register a *user* by global identifier. If it exists, add the calling participant to the user's participant list. Participant taken from the token. Idempotent. |
| US-UM-008 | Register user identifiers in bulk | **Rewrite** | Same semantics, batched. Per-entry result reporting (created / already-linked / rejected), partial success allowed. |
| US-UM-009 | Search for UserIdentifier | **Merge into US-UM-006** | The "does this identifier exist for this participant" question is a filter on the user search. |
| US-UM-010 | Inject UserIdentifier IDs into user | **Remove** | Artefact of the federation model. |
| US-UM-011 | Update UserIdentifier | **Revise** | Becomes "update the participant-local identifier on a user↔participant link" (only if D5 is accepted; otherwise remove). |
| US-UM-012 | Delete UserIdentifier | **Rewrite** | Becomes **"Unlink a user from my organization"** — removes the link, never the user. Deleting the user is a separate, data-subject-initiated concern (GDPR Art. 17) and is currently not covered by any story at all — see US-UM-014. |
| US-UM-013 | Cross-participant federation | **Remove** | Confirmed: no longer required. The deterministic rule in Section 5.3 replaces it. |
| **US-UM-014** | *(new)* Erase my user record | **New** | Should-have. A data subject asks for deletion; consents are retained in anonymised/audit form or deleted per a documented policy. A consent manager without an erasure story is an awkward gap. |

### 7.3 Participant Management (9 → 8)

| ID | Title | Action | Note |
|----|-------|--------|------|
| US-PM-001 | Register participant | **Rewrite** | Identifier taken from the token (never the body). Optional legal name, self-description URI, email, contact endpoints. No client secret, no DSC bootstrap. |
| US-PM-002 | Participant login | **Rewrite** | **"Authenticate as a participant with an external IDP token"** — keep the story (the validation criteria are needed), delete the endpoint. |
| US-PM-003 | List all participants | Revise | Response includes the global participant identifier, as requested. No secrets to exclude any more. |
| US-PM-004 | Get participant by ID | Revise | Lookup by global identifier instead of clientID. |
| US-PM-005 | View my participant profile | Keep | Fixes bug 12.1 by construction — there are no secrets to expose. |
| US-PM-006 | Update participant endpoints | Revise | Endpoint set reduced to contact/notification endpoints. Data-exchange endpoints removed. |
| US-PM-007 | Delete my participant | Revise | Add explicit cascade semantics: links removed, consents retained (audit), privacy notices archived. |
| US-PM-008 | Retrieve consent signature public key | **Rewrite** | JWKS at `/.well-known/jwks.json` with `kid` and rotation support, instead of a single PEM. |
| US-PM-009 | Configure DSC integration | **Remove** | As requested. |

### 7.4 Data Exchange (6 → 0)

All removed: US-DE-001 … US-DE-006. Document 06 deleted, ticket 012 deleted.

### 7.5 Supporting Modules (12 → 7)

| ID | Title | Action | Note |
|----|-------|--------|------|
| US-SM-001 | Consent Agent preferences | **Remove** | D4 — no JVM path for `contract-agent`. |
| US-SM-002 | Consent recommendations | **Remove** | D4. |
| US-SM-003 | Data exchange recommendations | **Remove** | Data exchange. |
| US-SM-004 | Consent verification email | **Rewrite** | Becomes **"Notify a user of a pending consent request"**. No verification token, no identity role — purely a notification. Demoted to Should-have. |
| US-SM-005 | API documentation | Revise | Generated from annotations. |
| US-SM-006 | Health check | Revise | Liveness + readiness. |
| US-SM-007 | Validate request payloads | **Remove as a story** | **Answering your question:** it is not a user story — it is a technical requirement dressed as one. Its intent is "no malformed payload ever reaches business logic; schema violations return 400 with a useful message." In Express that needed hand-written Joi middleware, so someone made it a work item. In Micronaut it is `@Valid` on a DTO plus a `ConstraintViolationException` handler — a line of configuration, not a feature. Proposal: delete the story and fold it into the API/error-handling ticket as acceptance criteria ("all request DTOs are annotated; violations produce RFC 7807 400 responses listing the offending fields"). |
| US-SM-008 | Consistent error handling | Revise | Recommend RFC 7807 Problem Details. |
| US-SM-009 | Logging | **Rewrite** | Structured JSON to stdout; no file rotation. Add a criterion on *not* logging token contents or personal identifiers at INFO. |
| US-SM-010 | OAuth authorization (future) | **Rewrite** | Becomes **"Endpoint-level authorization from IDP roles"**: every endpoint declares required roles; unauthenticated → 401, authenticated-but-unauthorized → 403; the role model is documented and extensible to RBAC/ABAC on resource attributes later. Promoted from Could-have to **Must-have** — with external auth this is the only thing standing between a participant token and another participant's data. |
| US-SM-011 | JSON-LD self-descriptions | **Revise / narrow** | Drop user self-descriptions (no signup, external identity). Keep the participant self-description as a stored opaque URI. Could-have. |
| US-SM-012 | Server sessions | **Remove** | **Confirmed — your reading is right.** Sessions existed for two reasons: caching auth between requests (now: stateless token validation) and the browser-facing consent views (no frontend in scope). Cookies would also force CSRF handling and the `SameSite=None` production configuration, all of which disappears. Removing sessions also removes the MongoDB-replica-set requirement, which was only ever there for `connect-mongo`. |

### 7.6 New Identity Stories (8 new)

| ID | Title | Priority |
|----|-------|----------|
| US-ID-001 | Register a trusted IDP by its `openid-configuration` endpoint | Must-have |
| US-ID-002 | Validate bearer tokens (issuer, signature via JWKS, audience, lifetime) | Must-have |
| US-ID-003 | Publish the documented token contract for users and participants | Must-have |
| US-ID-004 | Map IDP claims to Consent Manager roles (`USER`, `PARTICIPANT`, `CATALOG`) | Must-have |
| US-ID-005 | Just-in-time provisioning of user records from token claims | Must-have |
| US-ID-006 | Rotate/refresh IDP signing keys without restart | Should-have |
| US-ID-007 | Support multiple simultaneously trusted IDPs | Should-have |
| US-ID-008 | Reject tokens from unregistered issuers with a clear, non-leaking error | Must-have |

*(A runtime IDP-administration story was considered and dropped with D1 — the registry is static configuration.)*

### 7.7 Resulting Totals

| Module | Before | After |
|--------|--------|-------|
| Consent Management | 24 | 20 |
| User Management | 13 | 10 |
| Participant Management | 9 | 8 |
| Identity & Access *(new)* | 0 | 8 |
| Data Exchange | 6 | 0 |
| Supporting Modules | 12 | 7 |
| **Total** | **64** | **53** |

(Of the 53: ~23 carried over with edits, ~20 rewritten in substance, 10 new — 8 identity stories plus US-CM-025 and US-UM-014.)

---

## 8. Ticket Restructure

| Ticket | Action | Note |
|--------|--------|------|
| 001 Project Setup | **Rewrite** | Maven, Micronaut 4, Java 21, PostgreSQL + Flyway, Testcontainers, Logback JSON, OpenAPI Generator (API-first). |
| 002 Data Models | **Rewrite** | Entities + `V1__init.sql` migration + repositories. |
| 003 Auth Middleware | **Rewrite → "Identity & Access"** | Absorbs ticket 016. Covers US-ID-001…009, US-UM-002, US-PM-002, US-SM-010. Critical path. |
| 004 User Management | **Rewrite** | Absorbs ticket 006. |
| 005 Participant Management | **Rewrite** | DSC integration removed. |
| 006 UserIdentifier Management | **Delete** | Merged into 004. |
| 007 Privacy Notice Generation | **Revise** | Port; swap Axios client for `@Client`; plain URI parameters. |
| 008 Consent Grant Flow | **Revise, simplified** | One grant path + participant consent request. Complexity XL → L: the email reconciliation cases and inline exchange were most of the weight. |
| 009 Consent Lifecycle | **Revise** | Table-driven transitions. |
| 010 Consent Querying | **Revise** | Promoted — primary participant integration point. |
| 011 Consent Encryption | **Rewrite → "Consent Signing & Key Management"** | JWS receipts + JWKS endpoint. Complexity M → S/M. |
| 012 Data Exchange | **Delete** | |
| 013 Email Service | **Revise, shrunk** | Notification only. Medium → Low priority. |
| 014 Consent Agent Integration | **Delete** | D4 — no JVM path for `contract-agent`. |
| 015 API Docs & Infrastructure | **Rewrite** | Spec generated; management endpoints; Micrometer. |
| 016 OAuth Module | **Delete** | |
| 017 PDI Integration | **Delete** | The redirect is ~10 lines; if US-CM-014 is kept at all, it folds into ticket 010. |
| 018 Deployment | **Rewrite** | JVM/distroless image, optional native image, PostgreSQL in Helm, no replica set. |
| 019 Testing | **Rewrite** | JUnit 5 parameterized, Micronaut Test, Testcontainers (PostgreSQL + Keycloak), WireMock for the contract service. |
| **020** *(new)* | **Participant Notification / Outbox** | Covers US-CM-025. Should-have. |

### Revised Dependency Order

```
Phase 1  001 Project Setup
Phase 2  002 Data Models · 011 Consent Signing · 015 API Docs & Infra · 018 Deployment
Phase 3  003 Identity & Access                       [critical path]
Phase 4  004 User Management · 005 Participant Management
Phase 5  007 Privacy Notice Generation
Phase 6  008 Consent Grant Flow
Phase 7  009 Consent Lifecycle · 010 Consent Querying · 013 Notifications · 020 Outbox
Phase 9  019 Testing (continuous; formal gate after Phase 7)
```

15 tickets, 8 phases. The removal of the data-exchange phase and the identity-federation phase takes
two full phases out of the critical path.

---

## 9. Known-Bug Re-triage

The plan carries 40 catalogued bugs from the original implementation. After the redesign:

| Category | Count | Disposition |
|----------|-------|-------------|
| Mongoose/Express artefacts (missing `await` on `save()`, ObjectId reference comparison, stale-data returns on update, missing unique index on email, session key typo, duplicate token helper) | ~14 | **Void** — the target stack makes them impossible (transactions, unique constraints, immutable DTOs). Record them as "resolved by design" rather than deleting, so reviewers can see they were considered. |
| Auth/session bugs (unauthenticated email-verification endpoint, plaintext `clientSecret`, secrets exposed by `getMyParticipant`, unauthenticated pull exchange) | ~5 | **Void** — the affected features are removed. |
| Data-exchange bugs (`sendConsent` stub, swallowed encryption errors, suppressed consumer errors, participant query bug) | ~5 | **Void** — feature removed. |
| Genuine logic bugs that must not be reproduced (refused-but-granted in `giveConsentUser`, wrong parent field on re-grant, global count in `getUserConsents`, `toReceipt()` non-existent field, `resumeConsent` not persisting before use, no cascade on participant deletion, simulated-contract filtering, hardcoded API prefix in JSON-LD) | ~10 | **Carry forward** as explicit acceptance criteria and regression tests in the corresponding tickets. |
| Operational (key files loaded at import, simulated router escaping the error handler, TLS verification disabled, critical log that sends no email, empty template registry) | ~6 | **Carry forward** as configuration/robustness criteria. |

---

## 10. Consequences and Risks

1. **Participants must map their internal user IDs to the global identifier themselves.** The old
   `UserIdentifier` existed precisely to hold that mapping. Removing it moves the burden outward.
   D5 (optional `local_identifier` column) softens this but does not make it a lookup key.
2. **Global identifier uniqueness is asserted, not verified.** *(referenced by D6)* If two participants register the same
   email for genuinely different people, or the same person under an email and a DID, the Consent
   Manager cannot tell. The rule "never merge, never guess" is the right call, but the data space
   needs an identifier policy, and the documentation should say so explicitly rather than imply the
   Consent Manager resolves it.
3. **Removing data exchange removes the enforcement point.** Previously the Consent Manager was
   in the path of the exchange and could refuse to forward. Now it only records. The query API
   (US-CM-013) becomes the sole control, and it is advisory — participants must be *required* by
   data space governance to check it. This should be stated as an explicit non-goal in document 01
   so it is not mistaken for a gap.
4. **Effective revocation needs notification.** Recording a revocation that no participant learns
   about is weak. US-CM-025 / ticket 020 addresses this; if it is dropped, the limitation should be
   documented.
5. **Role mapping is the new security boundary.** Every misconfigured claim mapping is a potential
   cross-participant data leak. This argues for making the IDP registry static configuration (D1)
   and for the authorization tests in ticket 019 to be exhaustive and parameterized.
6. **Consent-agent capability is lost, not ported** (D4). Dropping `contract-agent` removes
   preference-driven recommendations from the product. If that capability is wanted later, it
   returns as a separate service behind an HTTP interface, not as an embedded library.
7. **The contract service dependency remains.** Privacy notices are still generated from contracts,
   so this coupling is unchanged and remains the largest external dependency.

---

## 11. Execution Plan

Assuming the decisions in Section 2 are made, the rewrite proceeds as step branches off
`ticket-61/work`, one PR each:

| Step | Scope | Status |
|------|-------|--------|
| 6 | `tickets/` — delete 5, add 1, rewrite 9, revise 4, plus `tickets/README.md` | **Done** |
| 1 | `01-architecture-overview.md` + `02-data-models.md` (technology and schema) | Pending |
| 2 | New `06-authentication-and-identity.md`; delete `06-data-exchange.md` | Pending |
| 3 | `04-user-management.md` + `05-participant-management.md` | Pending |
| 4 | `03-consent-management.md` + `07-supporting-modules.md` | Pending |
| 5 | `08-user-stories.md` (full disposition applied) | Pending |
| 7 | `README.md` index + `RELEASE_NOTES_<TOPIC>.md` | Pending |

Step 6 was taken first: the tickets are the artefact the implementation work is actually scheduled
from, and revising them surfaced the design decisions the narrative documents now have to match.
Documents 01-08 remain pre-revision until their steps land, and the index says so.
