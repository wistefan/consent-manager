# Reimplementation Tickets Index

Individual reimplementation tickets for the Prometheus-X Consent Manager. Each ticket groups related
user stories into an implementable work package with dependencies, acceptance criteria and a
complexity estimate.

These tickets reflect the scope and technology revision recorded in
[00-revision-proposal.md](../00-revision-proposal.md): **Java 21 + Micronaut 4 + PostgreSQL 16**,
authentication delegated to an external OIDC provider, and data exchange and user federation removed
from scope.

## Cross-Cutting Conventions

**Java only, and no build language at all.** The implementation language is Java 21. The build is
**Maven**: `pom.xml` is declarative configuration, not a script, so there is no build DSL to choose,
no build logic to review, and no second language in the repository. No Gradle, no Kotlin, no Groovy.
Code samples in these tickets are illustrative.

**The API is specification-first.** `api/openapi.yaml` is authored by hand and reviewed as the
design artefact; the Java API interfaces and request/response models are generated from it before
compilation, and controllers implement the generated interfaces. No ticket hand-writes a DTO, and
where a ticket lists `api/components/schemas/*.yaml` those are the authored source for what later
becomes generated Java. Validation constraints and the RFC 7807 error contract are declared once, in
the schema. TICKET-015 owns the specification's structure, linting and serving; TICKET-001 owns the
generation step.

## Execution Order

Tickets at the same phase can be parallelised. TICKET-003 is the critical path — nothing downstream
can be built or tested without it.

### Phase 1: Foundation

| Ticket | Title | Priority | Complexity | Dependencies |
|--------|-------|----------|------------|--------------|
| [TICKET-001](TICKET-001-project-setup.md) | Project Setup and Scaffolding | Critical | L | None |

### Phase 2: Core Infrastructure

| Ticket | Title | Priority | Complexity | Dependencies |
|--------|-------|----------|------------|--------------|
| [TICKET-002](TICKET-002-data-models.md) | Data Models and Schema | Critical | L | TICKET-001 |
| [TICKET-011](TICKET-011-consent-signing.md) | Consent Signing and Key Management | High | S/M | TICKET-001 |
| [TICKET-015](TICKET-015-api-docs-and-infrastructure.md) | API Docs and Cross-Cutting Infrastructure | High | M | TICKET-001 |
| [TICKET-018](TICKET-018-deployment.md) | Deployment Configuration | Medium | M | TICKET-001 |

### Phase 3: Identity (Critical Path)

| Ticket | Title | Priority | Complexity | Dependencies |
|--------|-------|----------|------------|--------------|
| [TICKET-003](TICKET-003-identity-and-access.md) | Identity and Access | Critical | L | TICKET-001, TICKET-002 |

### Phase 4: Entity Management

| Ticket | Title | Priority | Complexity | Dependencies |
|--------|-------|----------|------------|--------------|
| [TICKET-004](TICKET-004-user-management.md) | User Management | Critical | M | TICKET-002, TICKET-003 |
| [TICKET-005](TICKET-005-participant-management.md) | Participant Management | Critical | M | TICKET-002, TICKET-003 |

### Phase 5: Privacy Notices

| Ticket | Title | Priority | Complexity | Dependencies |
|--------|-------|----------|------------|--------------|
| [TICKET-007](TICKET-007-privacy-notice-generation.md) | Privacy Notice Generation | High | M | TICKET-002, TICKET-003, TICKET-005 |

### Phase 6: Consent Core

| Ticket | Title | Priority | Complexity | Dependencies |
|--------|-------|----------|------------|--------------|
| [TICKET-008](TICKET-008-consent-grant-flow.md) | Consent Grant Flow | Critical | L | TICKET-002, TICKET-003, TICKET-007, TICKET-011 |

### Phase 7: Consent Extensions

| Ticket | Title | Priority | Complexity | Dependencies |
|--------|-------|----------|------------|--------------|
| [TICKET-009](TICKET-009-consent-lifecycle.md) | Consent Lifecycle Operations | High | M | TICKET-008 |
| [TICKET-010](TICKET-010-consent-querying.md) | Consent Querying | **Critical** | M | TICKET-008 |
| [TICKET-020](TICKET-020-participant-notifications.md) | Participant Consent Notifications | High | M | TICKET-005, TICKET-008, TICKET-009 |
| [TICKET-013](TICKET-013-notification-service.md) | User Notification Service | Low | S | TICKET-001 |

### Phase 8: Quality Assurance (Continuous; Formal Gate After Phase 7)

| Ticket | Title | Priority | Complexity | Dependencies |
|--------|-------|----------|------------|--------------|
| [TICKET-019](TICKET-019-testing.md) | Testing | High | L | TICKET-002, TICKET-003 (minimum); all tickets for full coverage |

## Summary Statistics

| Metric | Before Revision | After Revision |
|--------|-----------------|----------------|
| Total tickets | 19 | 15 |
| Critical priority | 7 | 7 |
| High priority | 7 | 6 |
| Medium priority | 3 | 1 |
| Low priority | 2 | 1 |
| User stories covered | 64 | 53 |
| Phases | 9 | 8 |
| XL-complexity tickets | 3 | 0 |

## Changes from the Pre-Revision Ticket Set

### Deleted

| Ticket | Reason |
|--------|--------|
| TICKET-006 — UserIdentifier Management | The `UserIdentifier` entity and cross-participant federation are removed. A user has one globally unique identifier and a set of participant links, governed by a single deterministic rule now owned by TICKET-004. |
| TICKET-012 — Data Exchange | Data exchange is out of scope. The transfer itself is executed between participants; the Consent Manager's role ends at producing a verifiable, queryable consent record. |
| TICKET-014 — Consent Agent Integration | The `contract-agent` package is Node-only and has no JVM path. Preference-driven recommendations are a separate product concern; nothing else in the plan depended on them. |
| TICKET-016 — OAuth Module | The Consent Manager is a resource server, not an authorization server. Token issuance belongs to the external IDP (TICKET-003). |
| TICKET-017 — PDI Integration | The redirect was a thin frontend concern and the service has no frontend. If it is wanted, it folds into TICKET-010. |

### Renamed and Repurposed

| Was | Is | Change |
|-----|----|--------|
| TICKET-003 — Authentication Middleware | **Identity and Access** | Four bespoke auth mechanisms, sessions and passwords replaced by one OIDC resource-server mechanism with a documented token contract. Absorbs TICKET-016. |
| TICKET-011 — Consent Encryption | **Consent Signing and Key Management** | AES+RSA payload encryption existed to protect transit to connectors; with that gone, what retains value is JWS receipt signing plus a JWKS endpoint. |
| TICKET-013 — Email Service | **User Notification Service** | Email no longer carries an identity-verification role. Two informational messages remain, best-effort. |

### Added

| Ticket | Reason |
|--------|--------|
| TICKET-020 — Participant Consent Notifications | Removing data exchange removed the enforcement point. A revocation nobody is told about is not an effective revocation, so consent state changes are pushed to the bound participants over a transactional outbox. |

### Rescoped

- **API-first code generation adopted** across the set (decision D7). The pre-revision plan
  maintained `docs/swagger.json` by hand alongside the code; an earlier draft of this revision
  proposed deriving the spec from annotations. Neither holds: the specification is authored and the
  code generated from it, which makes the contract reviewable before implementation and makes drift
  a compile error rather than a review obligation.


- **TICKET-010 (Consent Querying) promoted to Critical.** With the service out of the data path, the
  query API is the sole control point by which a participant establishes whether processing is
  currently permitted. `GET /consents/check` is added as the explicit replacement for the removed
  data-exchange gate.
- **TICKET-008 (Consent Grant Flow) reduced from XL to L.** Two grant flows became one, the four
  email-reconciliation cases are gone, and inline data exchange is gone.
- **TICKET-019 (Testing) reduced from XL to L**, and TICKET-011 from M to S/M, TICKET-009 from L
  to M, TICKET-013 from M to S. No XL-complexity tickets remain.

### Known-Bug Disposition

The pre-revision plan carried 40 catalogued bugs from the original implementation. After the
redesign:

| Category | Count | Disposition |
|----------|-------|-------------|
| Mongoose/Express artefacts — missing `await`, ObjectId reference comparison, stale-data returns, missing unique index, session key typo, duplicate token helper | ~14 | **Void by design.** Recorded in each ticket's "Known Issues Resolved by Design" table so reviewers can see they were considered rather than dropped. |
| Auth and session bugs — unauthenticated email verification, plaintext `clientSecret`, secrets exposed on profile, unauthenticated pull exchange | ~5 | **Void** — the affected features are removed |
| Data exchange bugs — `sendConsent` stub, swallowed encryption errors, suppressed consumer errors, participant query bug | ~5 | **Void** — feature removed |
| Genuine logic bugs | ~10 | **Carried forward** as explicit acceptance criteria and regression tests in the owning ticket |
| Operational robustness — eager key loading, error-handler coverage, TLS verification, template registry | ~6 | **Carried forward** as configuration and robustness criteria |

## Priority Definitions

- **Critical**: the system does not function without it; blocks other critical tickets.
- **High**: core functionality that users and participants depend on daily.
- **Medium**: important, not blocking.
- **Low**: convenience or operational polish.

## Complexity Definitions

- **S (Small)**: 1–2 days, a single endpoint or utility, minimal integration.
- **M (Medium)**: 3–5 days, several endpoints or a self-contained module, moderate integration.
- **L (Large)**: 1–2 weeks, multiple related endpoints with complex logic, significant integration.
- **XL (Extra Large)**: 2–3 weeks. No ticket in the revised set carries this estimate.
