# Reimplementation Tickets Index

This directory contains individual reimplementation tickets for the Prometheus-X Consent Manager. Each ticket groups related user stories into an implementable work package with dependencies, acceptance criteria, and complexity estimates.

## Execution Order

Tickets are ordered by dependency chain. Critical-path tickets must be completed first; tickets at the same dependency level can be parallelized.

### Phase 1: Foundation (No Dependencies)

| Ticket | Title | Priority | Complexity | Dependencies |
|--------|-------|----------|------------|--------------|
| [TICKET-001](TICKET-001-project-setup.md) | Project Setup and Scaffolding | Critical | L | None |

### Phase 2: Core Infrastructure (Depends on Phase 1)

| Ticket | Title | Priority | Complexity | Dependencies |
|--------|-------|----------|------------|--------------|
| [TICKET-002](TICKET-002-data-models.md) | Data Models | Critical | L | TICKET-001 |
| [TICKET-011](TICKET-011-consent-encryption.md) | Consent Encryption and Key Management | High | M | TICKET-001 |
| [TICKET-013](TICKET-013-email-service.md) | Email Service | Medium | M | TICKET-001 |
| [TICKET-015](TICKET-015-api-docs-and-infrastructure.md) | API Docs and Infrastructure | High | M | TICKET-001 |
| [TICKET-018](TICKET-018-deployment.md) | Deployment Configuration | Medium | M | TICKET-001 |

### Phase 3: Authentication and Session (Depends on Phase 2)

| Ticket | Title | Priority | Complexity | Dependencies |
|--------|-------|----------|------------|--------------|
| [TICKET-003](TICKET-003-auth-middleware.md) | Authentication Middleware | Critical | L | TICKET-001, TICKET-002 |

### Phase 4: Entity Management (Depends on Phase 3)

| Ticket | Title | Priority | Complexity | Dependencies |
|--------|-------|----------|------------|--------------|
| [TICKET-004](TICKET-004-user-management.md) | User Management | Critical | M | TICKET-002, TICKET-003 |
| [TICKET-005](TICKET-005-participant-management.md) | Participant Management | Critical | M | TICKET-002, TICKET-003 |
| [TICKET-014](TICKET-014-consent-agent-integration.md) | Consent Agent Integration | Low | M | TICKET-001, TICKET-003 |
| [TICKET-016](TICKET-016-oauth-module.md) | OAuth Module | Low | M | TICKET-003 |

### Phase 5: Identity Federation (Depends on Phase 4)

| Ticket | Title | Priority | Complexity | Dependencies |
|--------|-------|----------|------------|--------------|
| [TICKET-006](TICKET-006-user-identifier-management.md) | UserIdentifier Management | Critical | L | TICKET-002, TICKET-003, TICKET-004, TICKET-005 |
| [TICKET-007](TICKET-007-privacy-notice-generation.md) | Privacy Notice Generation | High | M | TICKET-002, TICKET-003, TICKET-005 |

### Phase 6: Consent Core (Depends on Phase 5)

| Ticket | Title | Priority | Complexity | Dependencies |
|--------|-------|----------|------------|--------------|
| [TICKET-008](TICKET-008-consent-grant-flow.md) | Consent Grant Flow | Critical | XL | TICKET-003, TICKET-006, TICKET-007, TICKET-011, TICKET-013 |

### Phase 7: Consent Extensions (Depends on Phase 6)

| Ticket | Title | Priority | Complexity | Dependencies |
|--------|-------|----------|------------|--------------|
| [TICKET-009](TICKET-009-consent-lifecycle.md) | Consent Lifecycle Operations | High | L | TICKET-008 |
| [TICKET-010](TICKET-010-consent-querying.md) | Consent Querying | High | M | TICKET-008 |
| [TICKET-017](TICKET-017-pdi-integration.md) | PDI Integration | Medium | S | TICKET-008 |

### Phase 8: Data Exchange (Depends on Phase 6)

| Ticket | Title | Priority | Complexity | Dependencies |
|--------|-------|----------|------------|--------------|
| [TICKET-012](TICKET-012-data-exchange.md) | Data Exchange | High | XL | TICKET-005, TICKET-008, TICKET-011 |

### Phase 9: Quality Assurance (Ongoing, Formal After Phase 7)

| Ticket | Title | Priority | Complexity | Dependencies |
|--------|-------|----------|------------|--------------|
| [TICKET-019](TICKET-019-testing.md) | Testing | High | XL | TICKET-002, TICKET-003 (minimum); all tickets for full coverage |

## Summary Statistics

| Metric | Value |
|--------|-------|
| Total tickets | 19 |
| Critical priority | 7 |
| High priority | 7 |
| Medium priority | 3 |
| Low priority | 2 |
| Total user stories covered | 64 |
| Total known bugs to address | 40 |
| Estimated phases | 9 |

## Priority Definitions

- **Critical**: Must be implemented for the system to function at all. Blocks other critical tickets.
- **High**: Core functionality that users and participants depend on daily.
- **Medium**: Important features that enhance usability but are not blocking.
- **Low**: Nice-to-have features, dead-code rewrites, or future enhancements.

## Complexity Definitions

- **S (Small)**: 1-2 days, single endpoint or utility, minimal integration.
- **M (Medium)**: 3-5 days, multiple endpoints or a self-contained module, moderate integration.
- **L (Large)**: 1-2 weeks, multiple related endpoints with complex logic, significant integration.
- **XL (Extra Large)**: 2-3 weeks, highly complex logic with many edge cases, extensive integration and testing.
