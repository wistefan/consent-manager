# Implementation Plan: Feature Documentation of Consent-Manager

## Overview
Produce a comprehensive feature documentation of the Prometheus-X Consent Manager codebase to prepare for reimplementation. The deliverables are a set of markdown files in a `reimplementation/` folder covering: system architecture and data models, detailed feature documentation organized by module, user stories for every feature, and individual reimplementation tickets with acceptance criteria.

## Steps

### Step 1: Create reimplementation folder structure, architecture overview, and data model documentation

Create the `reimplementation/` directory with the foundational documents:

**Files to create:**
- `reimplementation/README.md` — Table of contents and navigation guide for all documentation
- `reimplementation/01-architecture-overview.md` — System architecture document covering:
  - High-level system diagram (text-based) showing Consent Manager, Contract Service, Data Space Connectors, PDI Frontend, Catalog, and email services
  - Technology stack summary (TypeScript, Express.js, MongoDB, JWT auth, etc.)
  - API prefix and versioning scheme (`/v1`)
  - Authentication mechanisms (4 types: User JWT/OAuth, User Key header, Participant JWT, Catalog Consent Key)
  - External service integrations (Contract Service, participant DSCs, Mailchimp/Mandrill, service offering endpoints)
  - Deployment model (Docker, Kubernetes/Helm)
  - Environment configuration reference (all env vars from `.env.sample`)
- `reimplementation/02-data-models.md` — Complete data model documentation covering all 5 Mongoose models:
  - **Consent** — All 30+ fields with types, required flags, defaults, validators; instance methods (`isValid()`, `toReceipt()`, `getParticipants()`, `getParticipantsIDs()`); consent state enum (pending, draft, granted, revoked, expired, terminated, refused); event structure; parent/child linking
  - **User** — Fields (firstName, lastName, email, password, identifiers, oauth, jsonld, schema_version); bcrypt pre-save hook; `validatePassword()` method
  - **Participant** — Fields (did, legalName, legalPerson nested object, selfDescriptionURL, email, dataspaceEndpoint, endpoints object, clientID, clientSecret, jsonld, schema_version)
  - **UserIdentifier** — Fields (attachedParticipant, email, identifier, url, user); role in cross-participant user federation
  - **PrivacyNotice** — Fields (contract, title, dataProvider, controllerDetails, purposes, categoriesOfData, data, recipients, internationalTransfers, retentionPeriod, piiPrincipalRights, archivedAt); archival lifecycle
  - Entity relationship diagram (text-based)

**Acceptance criteria:**
- All 5 models documented with every field, type, and constraint
- Architecture diagram shows all external service interactions
- All 4 authentication mechanisms described with flow details
- All environment variables listed with descriptions

### Step 2: Document consent management features (core module)

Create detailed feature documentation for the consent management module — the largest and most complex part of the system.

**Files to create:**
- `reimplementation/03-consent-management.md` — Covering:
  - **Consent Lifecycle State Machine**: All states (pending, draft, granted, revoked, expired, terminated, refused) and all valid transitions with triggers
  - **Privacy Notice Generation**: How privacy notices are derived from contracts fetched from the Contract Service; creation of new notices; archival of stale notices; privacy notice retrieval by ID with external data population (contract, data resources, recipients, purposes)
  - **Consent Grant Flow (User Key)**: `POST /consents` — full flow including privacy notice validation, provider/consumer user identifier resolution, user interaction scenarios, email-based validation, parent/child consent inheritance for re-grants
  - **Consent Grant Flow (User JWT)**: `POST /consents/user` — same as above but for JWT-authenticated users; note the known bug where event="refused" creates a granted consent
  - **Email Verification Flow**: `GET /consents/emailverification` — callback endpoint for email-based consent validation; 4 user-identity reconciliation cases (2.a, 2.b, 2.c2, 2.d)
  - **Consent Revocation**: `DELETE /consents/:id` — sets status to revoked, consented=false
  - **Consent Refusal**: `POST /consents/:consentId/refuse` — sets status to refused
  - **Consent Termination**: `POST /consents/:consentId/terminate` — sets status to terminated
  - **Consent Re-confirmation**: `POST /consents/:consentId/re-confirm` — adds re-confirmed event, optionally triggers data exchange
  - **Consent Resume**: `POST /consents/:consentId/resume` — transitions pending/draft to granted; creates UserIdentifier from provided data
  - **Consent Receipt Format**: ISO/IEC 29184-compliant consent receipt transformation; Kantara Consent Receipt generation (KI-CR-v1.1.0)
  - **Consent Querying**: `GET /consents/me`, `GET /consents/me/:id`, `GET /consents/participants/:userId/`, `GET /consents/participants/:userId/:id` — pagination, receipt format option, parent/child inclusion
  - **PDI Iframe Redirect**: `GET /consents/pdi/iframe` — redirect to PDI frontend with query params
  - **Available Exchanges**: `GET /consents/exchanges/user` and `GET /consents/exchanges/:as` — fetching bilateral and ecosystem contracts
  - **Token Management**: `POST /consents/:consentId/token` (attach provider token) and `POST /consents/:consentId/validate` (verify token)
  - **Consent Encryption**: AES-256-CBC payload encryption + RSA key encryption for secure consent transmission

**Acceptance criteria:**
- Every consent endpoint documented with method, path, auth requirement, request/response format
- State machine diagram with all transitions and triggering actions
- Email verification flow documented with all 4 reconciliation cases
- Encryption scheme documented (AES-256-CBC + RSA)
- Known bugs documented (giveConsentUser refused bug)

### Step 3: Document user management, participant management, data exchange, and supporting modules

Create detailed feature documentation for the remaining modules.

**Files to create:**
- `reimplementation/04-user-management.md` — Covering:
  - **User Signup**: `POST /users/signup` — creates User, generates OAuth tokens, creates JSON-LD self-description
  - **User Login**: `POST /users/login` — email/password auth, bcrypt validation, JWT + refresh token generation, session creation
  - **User Logout**: `DELETE /users/logout` — session destruction
  - **User Profile**: `GET /users/me` — returns authenticated user with fresh tokens
  - **User Lookup**: `GET /users/:userId` — public user lookup (no auth); `POST /users/search` — find by email (catalog key auth)
  - **UserIdentifier Registration**: `POST /users/register` (single) and `POST /users/registers` (bulk) — participant-authenticated; creates UserIdentifier linked to participant
  - **UserIdentifier Search**: `POST /users/identifier/search` — check existence of participant + user identifier + user for a self-description + email combination
  - **UserIdentifier Injection**: `POST /users/identifier` — attach UserIdentifier IDs to an existing User
  - **UserIdentifier Update/Delete**: `PUT /users/:id` and `DELETE /users/:id` — participant-authenticated
  - **Cross-Participant User Federation**: How `checkUserIdentifier` auto-links users across participants by email matching; `TryFindMatchingUser` utility

- `reimplementation/05-participant-management.md` — Covering:
  - **Participant Registration**: `POST /participants` — Joi validation, duplicate check by clientID, optional DSC configuration (login + push public key/URI)
  - **Participant Login**: `POST /participants/login` — clientID/clientSecret auth, JWT generation (1h expiry)
  - **Participant Retrieval**: `GET /participants` (paginated list), `GET /participants/:id`, `GET /participants/clientId/:clientId`, `GET /participants/me`
  - **Participant Update**: `PUT /participants/clientId/:clientId` — update endpoints (dataExport, dataImport, consentExport, consentImport) and dataspaceEndpoint
  - **Participant Deletion**: `DELETE /participants/me`
  - **Public Key Management**: `GET /participants/consent-signature` — returns consent signature public key for verification
  - **DSC Integration**: How participants' Data Space Connectors are configured with consent manager's public key and URI

- `reimplementation/06-data-exchange.md` — Covering:
  - **Data Exchange Architecture**: Three-party flow (Consent Manager, Provider DSC, Consumer DSC)
  - **User-Triggered Exchange**: `POST /consents/:consentId/data-exchange` and `POST /data-exchange/push/user` — user initiates exchange for a granted consent
  - **Provider-Triggered Exchange**: `POST /data-exchange/push` — provider pushes data exchange for a user's consent
  - **Inline Exchange**: Triggered during consent grant when `triggerDataExchange=true` query param is set
  - **Exchange Protocol**: Step-by-step flow — encrypt consent -> POST to provider consentExport -> provider returns dataExchangeId -> provider calls back with token -> forward to consumer consentImport -> consumer verifies token
  - **Pull Exchange**: `POST /data-exchange/pull` — not implemented (stub returning error)

- `reimplementation/07-supporting-modules.md` — Covering:
  - **OAuth Module**: Authorization and token endpoints (currently dead code — not mounted); scope definitions (userRead, userWrite, organizationRead, organizationWrite); `validateAccessToken` middleware (unused)
  - **Consent Agent Integration**: Profile endpoints from `contract-agent` package; user preferences CRUD; configurations management; preference matching; consent and data exchange recommendations
  - **Email Service**: Mandrill transactional email integration; Nodemailer SMTP fallback; HTML templates (consent validation, verification, test)
  - **Session Management**: express-session with MongoDB store (connect-mongo); 24h cookie; secure/SameSite in production
  - **Logging**: Winston logger with daily-rotate-file; configurable max files and max size
  - **Validation**: Joi schemas for participant creation/update; request body validation middleware
  - **Error Handling**: Global error handler (BadRequestError 400, NotFoundError 404, generic 500)
  - **JSON-LD Self-Descriptions**: Generation for users and participants
  - **Simulated Contract Service**: Development-mode contract service at `/contracts`
  - **API Documentation**: Swagger UI at `/docs` serving OpenAPI spec
  - **Health Check**: `GET /health` endpoint

**Acceptance criteria:**
- Every endpoint across all modules documented with method, path, auth, request/response
- Data exchange protocol documented step-by-step with encryption details
- Cross-participant user federation logic documented
- All supporting modules (email, session, logging, validation, error handling) documented
- Dead code and unused features clearly marked

### Step 4: Define user stories for all features

Create user stories organized by module, following standard format (As a [role], I want [action], so that [benefit]).

**Files to create:**
- `reimplementation/08-user-stories.md` — Organized by module with the following sections:
  - **Consent Management User Stories** (~20-25 stories covering):
    - Privacy notice generation and retrieval
    - Consent grant (both auth flows)
    - Email-based consent verification
    - Consent revocation, refusal, termination, re-confirmation, resume
    - Consent querying and receipt format
    - Available exchanges discovery
    - PDI iframe redirect
    - Consent encryption and token management
    - Parent/child consent inheritance
  - **User Management User Stories** (~10-12 stories covering):
    - Signup, login, logout, profile
    - UserIdentifier CRUD (single and bulk)
    - Cross-participant user federation
    - User search and lookup
  - **Participant Management User Stories** (~8-10 stories covering):
    - Registration with DSC configuration
    - Login and JWT auth
    - CRUD operations
    - Public key management
    - Endpoint configuration
  - **Data Exchange User Stories** (~5-7 stories covering):
    - User-triggered push exchange
    - Provider-triggered push exchange
    - Inline exchange on consent grant
    - Token verification
  - **Supporting Module User Stories** (~8-10 stories covering):
    - Consent agent preferences and recommendations
    - Email notifications
    - API documentation access
    - Health monitoring
    - OAuth authorization (for future implementation)

  Each user story must include:
  - Unique ID (e.g., US-CM-001 for Consent Management, US-UM-001 for User Management)
  - Title
  - Story text (As a... I want... So that...)
  - Acceptance criteria (testable conditions)
  - Priority (must-have, should-have, could-have) based on whether the feature is core to the system

**Acceptance criteria:**
- At least 50 user stories covering all documented features
- Each story has a unique ID, role, action, benefit, and acceptance criteria
- Stories are prioritized and organized by module
- Stories reference the feature documentation sections they relate to

### Step 5: Create individual reimplementation ticket files

Create individual markdown files for reimplementation tickets, grouping related user stories into implementable work packages.

**Files to create:**
- `reimplementation/tickets/README.md` — Index of all tickets with IDs, titles, priorities, and dependencies
- `reimplementation/tickets/TICKET-001-project-setup.md` — Project scaffolding (TypeScript, Express, MongoDB, Docker, CI)
- `reimplementation/tickets/TICKET-002-data-models.md` — Implement all 5 Mongoose models (Consent, User, Participant, UserIdentifier, PrivacyNotice)
- `reimplementation/tickets/TICKET-003-auth-middleware.md` — Implement all 4 authentication mechanisms and session management
- `reimplementation/tickets/TICKET-004-user-management.md` — User signup, login, logout, profile, search
- `reimplementation/tickets/TICKET-005-participant-management.md` — Participant registration, login, CRUD, DSC integration
- `reimplementation/tickets/TICKET-006-user-identifier-management.md` — UserIdentifier CRUD, cross-participant federation, bulk registration
- `reimplementation/tickets/TICKET-007-privacy-notice-generation.md` — Privacy notice creation from contracts, archival, retrieval with external data population
- `reimplementation/tickets/TICKET-008-consent-grant-flow.md` — Core consent grant logic (both auth flows), user interaction scenarios, email verification
- `reimplementation/tickets/TICKET-009-consent-lifecycle.md` — Revoke, refuse, terminate, re-confirm, resume operations; state machine enforcement
- `reimplementation/tickets/TICKET-010-consent-querying.md` — Consent listing, receipt format, parent/child inclusion, participant-scoped queries
- `reimplementation/tickets/TICKET-011-consent-encryption.md` — AES-256-CBC + RSA encryption for consent signing, key management
- `reimplementation/tickets/TICKET-012-data-exchange.md` — User-triggered, provider-triggered, and inline data exchange flows; token management
- `reimplementation/tickets/TICKET-013-email-service.md` — Mandrill + Nodemailer integration, HTML templates, consent validation emails
- `reimplementation/tickets/TICKET-014-consent-agent-integration.md` — Profile/preference/recommendation endpoints from contract-agent package
- `reimplementation/tickets/TICKET-015-api-docs-and-infrastructure.md` — Swagger/OpenAPI spec, health check, logging, error handling, Joi validation
- `reimplementation/tickets/TICKET-016-oauth-module.md` — OAuth authorization/token endpoints (currently dead code — implement properly)
- `reimplementation/tickets/TICKET-017-pdi-integration.md` — PDI iframe redirect, frontend integration points
- `reimplementation/tickets/TICKET-018-deployment.md` — Docker, docker-compose, Kubernetes/Helm templates, environment configuration
- `reimplementation/tickets/TICKET-019-testing.md` — Test suite setup, unit tests, integration tests, test fixtures

Each ticket file must include:
- Ticket ID and title
- Priority (critical, high, medium, low)
- Dependencies (which tickets must be completed first)
- Related user stories (IDs from Step 4)
- Description of work
- Technical details (specific files, functions, patterns to implement)
- Acceptance criteria (testable conditions)
- Estimated complexity (S/M/L/XL)
- Known issues to address (bugs from original, improvements to make)

**Acceptance criteria:**
- At least 19 tickets covering all features documented in Steps 2-3
- Each ticket has clear dependencies forming a valid execution order
- Each ticket references its related user stories
- Each ticket has specific, testable acceptance criteria
- Known bugs from original codebase are noted for correction in reimplementation
- Ticket index provides a clear overview of the full reimplementation scope
