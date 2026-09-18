# Architecture Overview

## 1. System Role in the Prometheus-X Ecosystem

The Consent Manager is a central service in the Prometheus-X data space ecosystem. It manages the full lifecycle of user consent for data exchanges between participants (data providers and data consumers). It acts as the trust anchor that:

- **Generates privacy notices** from contracts negotiated in the Contract Service
- **Captures and stores user consent** with cryptographic signing
- **Orchestrates data exchanges** between participant Data Space Connectors (DSCs) by forwarding encrypted consent proofs
- **Federates user identities** across multiple participants using the UserIdentifier mechanism
- **Provides a consent receipt** in ISO/IEC 29184 and Kantara formats for auditability

## 2. High-Level System Diagram

```
+---------------------+
|   PDI Frontend      |  Browser-based consent UI
|   (iframe/redirect) |  for end users
+----------+----------+
           |
           | iframe redirect / user interaction
           v
+---------------------+         +---------------------+
|                     |  REST   |                     |
|  Consent Manager    |<------->|  Contract Service   |
|  (this service)     |  HTTP   |  (external)         |
|                     |         +---------------------+
|  - Consent CRUD     |
|  - Privacy Notices   |         +---------------------+
|  - User Management  |  REST   |                     |
|  - Participant Mgmt |<------->|  Catalog Service    |
|  - Data Exchange    |  HTTP   |  (external)         |
|  - Identity Fed.    |         +---------------------+
|                     |
+---+------+------+---+
    |      |      |
    |      |      +------- Email (Mandrill / SMTP) -----> Email Provider
    |      |
    |      +-------------- Consent Export/Import -------> Provider DSC
    |                                                     Consumer DSC
    |
    +-------- MongoDB (replica set) ---------> Data Store
```

### Interactions Summary

| External System | Protocol | Direction | Purpose |
|-----------------|----------|-----------|---------|
| **Contract Service** | HTTP REST | Outbound | Fetch bilateral/ecosystem contracts, verify contract validity, retrieve service offerings |
| **Data Space Connectors (DSCs)** | HTTP REST | Outbound | Login to participant DSCs, register users, push encrypted consent for data export/import |
| **PDI Frontend** | HTTP Redirect | Outbound | Redirect users to the consent UI for granting/managing consent |
| **Catalog Service** | HTTP REST | Inbound | Catalog authenticates via `x-catalog-consent-key` header to search users |
| **Email Provider** | SMTP / Mandrill API | Outbound | Send consent validation and verification emails to users |
| **MongoDB** | MongoDB Wire Protocol | Bidirectional | Persistent storage for all data (requires replica set for sessions) |

## 3. Technology Stack

| Component | Technology | Version/Details |
|-----------|-----------|-----------------|
| **Language** | TypeScript | Compiled to JavaScript via `tsc` |
| **Runtime** | Node.js | — |
| **Framework** | Express.js | 4.18 |
| **Database** | MongoDB | 7+ (requires replica set) |
| **ODM** | Mongoose | 7.3 |
| **Package Manager** | pnpm | — |
| **Authentication** | JWT (`jsonwebtoken`), bcryptjs, express-session + connect-mongo | — |
| **Email** | `@mailchimp/mailchimp_transactional` (Mandrill), Nodemailer | — |
| **Logging** | Winston + `winston-daily-rotate-file` | — |
| **Validation** | Joi | — |
| **API Documentation** | Swagger UI Express (OpenAPI 3.0) | Spec at `docs/swagger.json` |
| **HTTP Client** | Axios + `axios-cache-interceptor` | Caching for contract service calls |
| **Consent Agent** | `contract-agent` package | Dynamically loaded from Prometheus-X contract-consent-agent repo |
| **View Engine** | EJS | For consent receipt HTML rendering |
| **Testing** | Mocha, Chai, Supertest, Sinon, nock | Runner: `ts-mocha`, Reporter: `mochawesome` |

## 4. API Prefix and Versioning

All API routes are mounted under a configurable prefix:

- **Environment variable:** `API_PREFIX`
- **Default value:** `/v1`
- **Effect:** All routes documented as `/consents`, `/users`, `/participants`, `/data-exchange` are actually served at `{API_PREFIX}/consents`, etc.
- **Exceptions:** The following routes are mounted at the root level (not behind `API_PREFIX`):
  - `GET /health` - Health check endpoint
  - `GET /docs` - Swagger UI documentation
  - `/contracts` - Simulated contract service (development only)

There is no built-in multi-version support. The prefix acts as a namespace rather than a true API versioning mechanism.

## 5. Authentication Mechanisms

The system uses four distinct authentication mechanisms, each implemented as Express middleware. An endpoint may require one or more of these.

### 5.1 Participant JWT (`verifyParticipantJWT`)

**Purpose:** Authenticates participant services (machine-to-machine) and participant admin users.

**Flow:**
1. Extract token from `Authorization: Bearer <token>` header, or fall back to `?participant=<token>` query parameter.
2. **Session shortcut:** If `req.session.userParticipant` already exists, skip verification and use the cached identity.
3. **Standard path (no `serviceKey` in payload):** Verify token using `JWT_SECRET_KEY` environment variable. Extract `sub` claim as participant ID.
4. **DSC path (`serviceKey` in payload):** Base64-decode the JWT payload. Look up the `Participant` by `clientID` matching the `serviceKey`. Verify the token using that participant's `clientSecret` as the signing key.

**Attaches to request:**
- `req.decodedToken` - Full JWT payload
- `req.userParticipant.id` - Participant MongoDB ObjectId
- `req.session.userParticipant` - Cached in session for subsequent requests

**Token generation:** Participants receive a JWT upon login (`POST /participants/login`) with `{ sub: participant._id }`, signed with `JWT_SECRET_KEY`, expiring in 1 hour.

### 5.2 User JWT (`verifyUserJWT`)

**Purpose:** Authenticates end users via multiple strategies (session, user key header, or OAuth token).

**Flow (tried in order):**
1. **Session shortcut:** If `req.session.user` exists, use `req.session.user.id`.
2. **User Key header:** If `x-user-key` header is present, look up `UserIdentifier` by that ID, then resolve to the linked `User` (by `identifiers` array membership or email match). Auto-links the identifier to the user if not already linked.
3. **OAuth Bearer token:** If `Authorization: Bearer <token>` is present, verify using `OAUTH_SECRET_KEY`. Extract `sub` claim as user ID.

**Attaches to request:**
- `req.user.id` - User MongoDB ObjectId
- `req.decodedToken` - Full JWT payload (OAuth path only)
- `req.session.user` - Cached in session

**Token generation:** Users receive an access token and refresh token upon login (`POST /users/login`) or signup (`POST /users/signup`), signed with `OAUTH_SECRET_KEY`, expiring in 1 hour.

### 5.3 User Key (`verifyUserKey`)

**Purpose:** Authenticates end users via a simple header containing their UserIdentifier ID. Used for consent grant operations where the calling participant identifies the user.

**Flow:**
1. Read the `x-user-key` header value.
2. Look up `UserIdentifier` by that ID.
3. Return 401 if not found.

**Attaches to request:**
- `req.userIdentifier.id` - UserIdentifier MongoDB ObjectId

**Note:** This is a simpler variant than `verifyUserJWT`'s user-key path. It only resolves the UserIdentifier, not the full User.

### 5.4 Catalog Consent Key (`consentKeyCheck`)

**Purpose:** Authenticates the Catalog Service for inter-service calls (e.g., user search).

**Flow:**
1. Read the `x-catalog-consent-key` header value.
2. Compare against `X_CATALOG_CONSENT_KEY` environment variable (strict equality).
3. Return 401 if missing or mismatched.

**Attaches to request:** Nothing - simply gates access.

### Authentication Summary by Endpoint Group

| Endpoint Group | Primary Auth | Notes |
|---------------|-------------|-------|
| `POST /consents` (grant) | `verifyUserKey` | Participant identifies user via header |
| `POST /consents/user` (grant) | `verifyUserJWT` | User authenticates directly |
| `GET /consents/me` | `verifyUserJWT` | User queries own consents |
| `GET /consents/participants/:userId` | `verifyParticipantJWT` | Participant queries user's consents |
| Consent lifecycle ops | `verifyParticipantJWT` | Revoke, refuse, terminate, resume, re-confirm |
| `POST /users/signup`, `POST /users/login` | None | Public endpoints |
| `GET /users/me` | `verifyUserJWT` | User profile |
| `POST /users/register` | `verifyParticipantJWT` | Participant registers user identifier |
| `POST /users/search` | `consentKeyCheck` | Catalog searches users |
| `POST /participants` | None | Public registration |
| `POST /participants/login` | None | Public login |
| `GET /participants/me` | `verifyParticipantJWT` | Participant profile |
| Data exchange endpoints | `verifyParticipantJWT` | Participant triggers exchange |

## 6. External Service Integrations

### 6.1 Contract Service

**Base URL:** Configured via `CONTRACT_SERVICE_BASE_URL` environment variable (default: `http://localhost:3000/contracts`).

**HTTP Client:** Axios with `axios-cache-interceptor` for response caching.

**Endpoints called:**

| Method | Path | Purpose |
|--------|------|---------|
| `GET` | `/bilaterals/for/{participantSD64}?hasSigned=true` | Fetch bilateral contracts signed by a participant |
| `GET` | `/contracts/for/{participantSD64}?hasSigned=true` | Fetch ecosystem contracts signed by a participant |
| `GET` | `/verify/{providerSD64}/{consumerSD64}` | Validate a contract between two parties |
| `GET` | `/bilaterals/{contractId}` | Fetch a specific bilateral contract by ID |
| `GET` | `/contracts/{contractId}` | Fetch a specific ecosystem contract by ID |

**Note:** `{participantSD64}` and `{providerSD64}/{consumerSD64}` are Base64-encoded self-description URLs of the participants.

The service also fetches arbitrary URLs from contract payloads to resolve service offering self-descriptions and software resource metadata.

**Development mode:** A simulated contract service router is mounted at `/contracts` in the Express app for local development without an external contract service.

### 6.2 Data Space Connectors (DSCs)

Each participant may configure a `dataspaceEndpoint` URL pointing to their DSC, plus four specific operation endpoints (`endpoints.dataExport`, `endpoints.dataImport`, `endpoints.consentExport`, `endpoints.consentImport`).

**Outbound calls to DSCs:**

| Method | Path | Purpose |
|--------|------|---------|
| `POST` | `{dataspaceEndpoint}/login` | Authenticate to participant's DSC using `clientID`/`clientSecret` |
| `POST` | `{dataspaceEndpoint}/private/users/register` | Register a user in the participant's DSC |
| `POST` | `{dataspaceEndpoint}/private/users/app` | Register user app-side with consent reference |
| `POST` | `{endpoints.consentExport}` | Send encrypted consent to data provider's DSC for export |
| `POST` | `{endpoints.consentImport}` | Send encrypted consent to data consumer's DSC for import |

### 6.3 Email Service

Two email providers are supported, configured independently:

**Mailchimp/Mandrill:**
- Activated when `MANDRILL_API_KEY` is set
- Uses `@mailchimp/mailchimp_transactional` SDK
- `MANDRILL_ENABLED=true` sends to real recipients; `false` redirects all email to `MANDRILL_FROM_EMAIL`
- Supports both Mandrill-hosted templates and local HTML templates
- Primary provider for consent validation emails

**Nodemailer (SMTP):**
- Activated when both `NODEMAILER_USER` and `NODEMAILER_PASS` are set
- Standard SMTP transport with TLS (`rejectUnauthorized: false` for development)
- Fallback/alternative email provider

### 6.4 Consent Agent (`contract-agent` Package)

- **Package:** `contract-agent` from the Prometheus-X contract-consent-agent repository
- **Activation:** Only initialized if `consent-agent.config.json` exists in the project root
- **Function:** Provides AI-powered consent and data exchange recommendations based on user preferences
- **Routes:** Dynamically mounted under `{API_PREFIX}/` (e.g., `/v1/profile`, `/v1/preferences`), protected by `verifyUserJWT`

## 7. Deployment Model

### 7.1 Docker

**Docker Compose** (`docker-compose.yml`) defines two services:
- `consent-manager` - The application container (built from `docker/app/`)
- `consent-manager-mongodb` - MongoDB 7 container with replica set initialization

The MongoDB container is configured as a single-node replica set (required for change streams and transactions used by express-session).

### 7.2 Kubernetes / Helm

A Helm chart is provided for Kubernetes deployment:

- **Chart.yaml** - Chart metadata (name, version, app version)
- **values.yaml** - Configurable values (image, replicas, ports, env vars, ingress)
- **templates/**:
  - `deployment.yaml` - Kubernetes Deployment with configurable replicas, resource limits, and environment injection from Secrets
  - `service.yaml` - Kubernetes Service (ClusterIP)
  - `secret.yaml` - Kubernetes Secret for sensitive environment variables
  - `namespace.yaml` - Namespace creation

### 7.3 Terraform

Basic Terraform configuration is provided in `terraform/` for infrastructure provisioning (provider configuration, variables, outputs).

## 8. Server Configuration

### 8.1 CORS

```
cors({ origin: true, credentials: true })
```
Allows all origins with credentials (cookies, authorization headers). This is permissive by design since the consent manager must accept requests from any participant's frontend.

### 8.2 Session Management

- **Store:** MongoDB-backed via `connect-mongo`
- **Cookie name:** Configurable via `SESSION_COOKIE_NAME` (default: `consentmanagersessid`)
- **Secret:** `SESSION_SECRET` environment variable
- **Cookie settings:**
  - Production: `sameSite: "none"`, `secure: true`
  - Development: `sameSite: "lax"`, `secure: false`
- **Max age:** 24 hours
- **Session ID generation:** `crypto.randomUUID()`
- **Resave:** `true`
- **Save uninitialized:** `false`

### 8.3 Request Processing

- **Body parsing:** `express.json()` (no size limit configured)
- **Proxy trust:** `trust proxy: true` (for running behind reverse proxies/load balancers)
- **HTTP logging:** Morgan middleware for request logging

## 9. Environment Configuration Reference

### Core Settings

| Variable | Default | Required | Description |
|----------|---------|----------|-------------|
| `NODE_ENV` | `development` | No | Runtime environment (`development` or `production`). Controls session cookie security, error verbosity, and URL construction. |
| `PORT` | `3000` | No | HTTP server listen port. |
| `APP_ENDPOINT` | `http://localhost:3000` | Yes | Public base URL of this service. Used in email callback URLs and self-description generation. |
| `API_PREFIX` | `/v1` | No | URL prefix for all API routes. |
| `APPLICATION_NAME` | `consentmanager-pdi` | No | Human-readable application name used in consent receipts. |
| `FEDERATED_APPLICATION_IDENTIFIER` | `http://localhost:3000` | No | Base identifier for this consent manager instance in federated context. |

### Database

| Variable | Default | Required | Description |
|----------|---------|----------|-------------|
| `MONGO_URI` | `mongodb://consent-manager-mongodb:27017/consent-manager` | Yes | MongoDB connection string. Must point to a replica set. |
| `MONGO_URI_TEST` | `mongodb://consent-manager-mongodb:27017/consent-manager-test` | No | MongoDB connection string for test environment. |

### Authentication & Security

| Variable | Default | Required | Description |
|----------|---------|----------|-------------|
| `JWT_SECRET_KEY` | `secret123` | Yes | Secret for signing and verifying participant JWT tokens. |
| `OAUTH_SECRET_KEY` | `abc123secret` | Yes | Secret for signing and verifying user OAuth access tokens. |
| `SESSION_SECRET` | `secret123` | Yes | Secret for signing session cookies. |
| `SESSION_COOKIE_NAME` | `consentmanagersessid` | No | Name of the session cookie. |
| `SALT_ROUNDS` | `10` | No | bcrypt salt rounds for password hashing. |
| `X_CATALOG_CONSENT_KEY` | *(empty)* | No | Shared secret for catalog-to-consent-manager authentication. |

### External Services

| Variable | Default | Required | Description |
|----------|---------|----------|-------------|
| `CONTRACT_SERVICE_BASE_URL` | `http://localhost:3000/contracts` | Yes | Base URL of the external Contract Service API. |
| `PDI_ENDPOINT` | `http://localhost:3331` | No | URL of the PDI (Personal Data Intermediary) frontend for user consent UI. |

### Email - Mandrill

| Variable | Default | Required | Description |
|----------|---------|----------|-------------|
| `MANDRILL_API_KEY` | `yourkey` | No | Mailchimp Transactional (Mandrill) API key. Set to enable Mandrill email. |
| `MANDRILL_ENABLED` | `false` | No | When `true`, emails are sent to real recipients. When `false`, all emails redirect to `MANDRILL_FROM_EMAIL`. |
| `MANDRILL_FROM_EMAIL` | `noreply@visionstrust.com` | No | Default sender email address for Mandrill. |
| `MANDRILL_FROM_NAME` | `noreply` | No | Default sender display name for Mandrill. |

### Email - SMTP (Nodemailer)

| Variable | Default | Required | Description |
|----------|---------|----------|-------------|
| `NODEMAILER_HOST` | *(empty)* | No | SMTP server hostname. |
| `NODEMAILER_PORT` | *(empty)* | No | SMTP server port. |
| `NODEMAILER_USER` | *(empty)* | No | SMTP authentication username. Both user and pass must be set to enable Nodemailer. |
| `NODEMAILER_PASS` | *(empty)* | No | SMTP authentication password. |

### Consent Receipt Fields

| Variable | Default | Required | Description |
|----------|---------|----------|-------------|
| `PRIVACY_RIGHTS` | *(empty)* | No | Comma-separated list of privacy rights included in consent receipts. |
| `WITHDRAWAL_METHOD` | *(empty)* | No | Description of consent withdrawal method for receipts. |
| `CODE_OF_CONDUCT` | *(empty)* | No | Code of conduct reference for consent receipts. |
| `IMPACT_ASSESSMENT` | *(empty)* | No | Impact assessment reference for consent receipts. |
| `AUTHORITY_PARTY` | *(empty)* | No | Authority party reference for consent receipts. |

### Logging

| Variable | Default | Required | Description |
|----------|---------|----------|-------------|
| `WINSTON_LOGS_MAX_FILES` | `14d` | No | Maximum age of log files before rotation/deletion. |
| `WINSTON_LOGS_MAX_SIZE` | `20m` | No | Maximum size of each log file before rotation. |

### Cryptographic Keys (File-Based)

These are not environment variables but file paths that must exist:

| File | Purpose |
|------|---------|
| `src/config/keys/consentSignature.pem` | RSA private key for signing consent payloads |
| `src/config/keys/consentSignaturePublic.pem` | RSA public key for consent signature verification (shared with participants) |
| `src/config/keys/AESKey.pem` | AES-256 key for encrypting consent data payloads |
