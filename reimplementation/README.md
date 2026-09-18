# Prometheus-X Consent Manager - Reimplementation Documentation

This folder contains comprehensive feature documentation of the Prometheus-X Consent Manager, prepared to guide a ground-up reimplementation of the system.

## Document Index

### Foundational Documents

| # | Document | Description |
|---|----------|-------------|
| 01 | [Architecture Overview](./01-architecture-overview.md) | System architecture, technology stack, authentication mechanisms, external integrations, deployment model, and environment configuration |
| 02 | [Data Models](./02-data-models.md) | Complete documentation of all 5 Mongoose data models with field-level detail, relationships, methods, and hooks |

### Feature Documentation (by Module)

| # | Document | Description |
|---|----------|-------------|
| 03 | [Consent Management](./03-consent-management.md) | Core consent lifecycle: state machine, privacy notices, grant flows, revocation, encryption, querying *(Step 2)* |
| 04 | [User Management](./04-user-management.md) | User signup/login, profile, UserIdentifier CRUD, cross-participant federation *(Step 3)* |
| 05 | [Participant Management](./05-participant-management.md) | Participant registration, login, CRUD, DSC integration *(Step 3)* |
| 06 | [Data Exchange](./06-data-exchange.md) | Data exchange architecture, user/provider-triggered flows, token protocol *(Step 3)* |
| 07 | [Supporting Modules](./07-supporting-modules.md) | OAuth, Consent Agent, email, session, logging, validation, error handling, API docs *(Step 3)* |

### Planning Documents

| # | Document | Description |
|---|----------|-------------|
| 08 | [User Stories](./08-user-stories.md) | User stories for all features, organized by module with acceptance criteria *(Step 4)* |
| 09 | [Reimplementation Tickets](./tickets/README.md) | Individual work tickets with dependencies, acceptance criteria, and estimates *(Step 5)* |

## How to Use This Documentation

1. **Start with the Architecture Overview** (01) to understand the system's role in the Prometheus-X ecosystem, its external dependencies, and deployment model.
2. **Review the Data Models** (02) to understand the core entities, their fields, and relationships.
3. **Read the Feature Documentation** (03-07) for detailed behavior of each module, including every API endpoint, business rules, and edge cases.
4. **Use the User Stories** (08) to understand the system from each actor's perspective and to validate that a reimplementation covers all use cases.
5. **Follow the Reimplementation Tickets** (09) for a structured, dependency-ordered plan to rebuild the system.

## Conventions

- **API paths** are shown without the configurable `API_PREFIX` (default `/v1`). The actual deployed path is `{API_PREFIX}{path}`, e.g., `/v1/consents`.
- **Auth requirements** are noted per endpoint using shorthand: `[UserJWT]`, `[UserKey]`, `[ParticipantJWT]`, `[CatalogKey]`, or `[None]`.
- **Known bugs** in the original implementation are called out in warning blocks so the reimplementation can correct them.
- **Dead code** (features present in source but not active) is clearly marked.
