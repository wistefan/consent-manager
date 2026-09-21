# Supporting Modules

This document provides comprehensive feature documentation for all supporting and infrastructure modules in the Consent Manager. It covers the OAuth module (dead code), Consent Agent integration, email services, session management, logging, validation, error handling, JSON-LD self-descriptions, the simulated contract service, API documentation, and the health check endpoint.

> **Source files:** `src/routes/oauth.ts`, `src/middleware/oauth.ts`, `src/libs/OAuth/`, `src/libs/emails/`, `src/middleware/session.ts`, `src/libs/loggers/`, `src/middleware/joiValidation.ts`, `src/libs/joi/`, `src/middleware/globalErrorHandler.ts`, `src/errors/`, `src/libs/jsonld/`, `src/simulated/contract/`, `src/server.ts`, `src/routes/index.ts`

---

## Table of Contents

1. [OAuth Module (Dead Code)](#1-oauth-module-dead-code)
2. [Consent Agent Integration](#2-consent-agent-integration)
3. [Email Service](#3-email-service)
4. [Session Management](#4-session-management)
5. [Logging](#5-logging)
6. [Request Validation (Joi)](#6-request-validation-joi)
7. [Error Handling](#7-error-handling)
8. [JSON-LD Self-Descriptions](#8-json-ld-self-descriptions)
9. [Simulated Contract Service](#9-simulated-contract-service)
10. [API Documentation (Swagger)](#10-api-documentation-swagger)
11. [Health Check](#11-health-check)
12. [Server Setup and Middleware Stack](#12-server-setup-and-middleware-stack)
13. [Known Bugs](#13-known-bugs)

---

## 1. OAuth Module (Dead Code)

> **⚠ Dead Code:** The OAuth routes are defined in `src/routes/oauth.ts` but are **not imported or mounted** in `src/routes/index.ts`. None of these endpoints are accessible at runtime.

### 1.1 OAuth Routes

**File:** `src/routes/oauth.ts`

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/authorize` | Render OAuth consent page |
| `POST` | `/token` | Exchange authorization code for tokens |

#### `GET /authorize`

**Logic:**
1. Check `req.session.user` -- if absent, redirect to `/users/login`.
2. Parse `scope` query parameter (space-delimited string).
3. Map each scope to its human-readable description from `OAUTH_SCOPES`.
4. Render the `consent.ejs` EJS template with `clientName` (from query) and `requestedScopes`.

**Query parameters:** `scope` (space-delimited scopes), `client_name` (display name of the requesting client)

#### `POST /token`

**Logic:**
1. Validate `req.body.code` against `req.session.authorizationCode`.
2. Look up User by `req.session.user.id`.
3. Validate requested scopes via `validateScopes(requestedScopes)`.
4. Generate access token (JWT, 1h expiry) and refresh token.
5. Return token response.

**Response:**
```json
{
  "access_token": "<JWT>",
  "token_type": "bearer",
  "expires_in": 3600,
  "refresh_token": "<hex string>"
}
```

### 1.2 OAuth Scopes

**File:** `src/libs/OAuth/scopes.ts`

| Scope Key | Description |
|-----------|-------------|
| `userRead` | Read user data |
| `userWrite` | Modify user data |
| `organizationRead` | Read organization data |
| `organizationWrite` | Modify organization data |

> **Note:** Despite being defined, these scopes are only used to generate access tokens during user signup/login. The scope values are never checked or enforced by any middleware.

### 1.3 OAuth Token Utilities

**File:** `src/libs/OAuth/tokens.ts`

| Function | Description |
|----------|-------------|
| `validateScopes(requestedScopes)` | Filters requested scopes against valid scope keys. Returns `null` if none are valid. |
| `generateAccessToken(user, scopes)` | Creates JWT signed with `OAUTH_SECRET_KEY`, payload `{ sub, email, scopes }`, 1h expiry. |
| `generateRefreshToken(user, { save })` | Generates random 32-byte hex string. Stores on `user.oauth.refreshToken`. Optionally saves to DB. |
| `generateNewAccessToken(user, scopes)` | Exact duplicate of `generateAccessToken`. |

> **Note:** `generateAccessToken` and `generateNewAccessToken` are identical functions. One should be removed.

### 1.4 OAuth Middleware (Unused)

**File:** `src/middleware/oauth.ts`

**Function:** `validateAccessToken`

**Logic:**
1. Extract bearer token from `Authorization` header or `access_token` query parameter.
2. Verify JWT against `process.env.OAUTH_SECRET_KEY`.
3. Set `req.user` to the decoded payload.
4. Has a TODO comment for scope and expiration validation.

**Status:** Not used by any mounted route. The `verifyUserJWT` middleware performs similar token verification and is the active user authentication mechanism.

---

## 2. Consent Agent Integration

### 2.1 Package

The `contract-agent` package is installed from the Prometheus-X organization:
```
https://gitpkg.now.sh/Prometheus-X-association/contract-consent-agent/contract-agent?v1.0.0
```

### 2.2 Server-Side Initialization

**File:** `src/server.ts`

During server startup:

1. Resolve the config file path (default: `../consent-agent.config.json` from `src/`, which resolves to the project root).
2. Check if the config file exists with `fs.existsSync`.
3. If present:
   - `Agent.setConfigPath(configPath, __filename)` -- sets the config file for the agent.
   - `Agent.setProfilesHost("profiles")` -- configures the profiles collection name.
   - `await ConsentAgent.retrieveService()` -- initializes the consent agent service.
4. If absent, the agent is silently skipped.

### 2.3 Route Mounting

**File:** `src/routes/index.ts`

```typescript
app.use(API_PREFIX + "/", verifyUserJWT, ConsentAgentRouter);
```

The `ConsentAgentRouter` (imported from `contract-agent`) is mounted at the API prefix root with `verifyUserJWT` authentication. The specific endpoints this router exposes are defined inside the `contract-agent` package and typically include:

- Profile management (user preference profiles)
- User preferences CRUD
- Configurations management
- Preference matching
- Consent and data exchange recommendations

### 2.4 Configuration

The consent agent configuration specifies MongoDB data sources for its recommendation engine.

**Test configuration** (`src/tests/mocks/consent-agent.config.json`):

```json
{
  "existingDataCheck": false,
  "dataProviderConfig": [
    { "source": "users", "url": "mongodb://...", "watchChanges": true },
    { "source": "consents", "url": "mongodb://...", "watchChanges": true },
    { "source": "privacynotices", "url": "mongodb://...", "watchChanges": true },
    { "source": "participants", "url": "mongodb://..." },
    { "source": "useridentifiers", "url": "mongodb://..." },
    { "source": "profiles", "url": "mongodb://...", "hostsProfiles": true }
  ]
}
```

- `existingDataCheck` -- Whether to verify existing data on startup.
- `watchChanges` -- Enables MongoDB change streams for real-time updates.
- `hostsProfiles: true` -- Marks the collection that stores user preference profiles.

> **Note:** No production `consent-agent.config.json` exists in the repository. The sample configuration (`consent-agent.config.sample.json`) must be copied and configured for each deployment.

---

## 3. Email Service

The Consent Manager supports two email providers: Mandrill (Mailchimp Transactional) and Nodemailer (SMTP). The provider is selected based on environment configuration.

### 3.1 Mandrill (Mailchimp Transactional)

**Files:** `src/libs/emails/mailchimp/MailchimpClient.ts`, `src/libs/emails/mailchimp/MailchimpEmailClient.ts`

**Configuration** (`src/config/mailchimpConfig.ts`):

| Env Variable | Description |
|-------------|-------------|
| `MANDRILL_API_KEY` | Mandrill API key (enables the service when set) |
| `MANDRILL_FROM_EMAIL` | Default sender email address |
| `MANDRILL_FROM_NAME` | Default sender display name |

**Singleton:** `MailchimpClient` -- initialized at module load. If `MANDRILL_API_KEY` is set, creates an active client; otherwise creates a deactivated instance (all methods no-op).

**Class: `MailchimpEmailClient`**

| Property | Type | Description |
|----------|------|-------------|
| `client` | ApiClient | Mandrill API client instance (or `null`) |
| `baseFromMail` | string | Default sender email |
| `baseFromName` | string | Default sender name |
| `activated` | boolean | Whether the client is active |

| Method | Description |
|--------|-------------|
| `sendMessage(options)` | Send email with provided `MessagesSendRequest` options. No-ops if deactivated. |
| `sendMessageFromLocalTemplate(options, templateName, variables)` | Inject HTML from a local template, then send via Mandrill. |
| `sendMessageFromTemplate(options, templateName, variables)` | Send using a Mandrill server-side template (via `sendTemplate`). |

### 3.2 Nodemailer (SMTP)

**Files:** `src/libs/emails/nodemailer/index.ts`, `src/libs/emails/nodemailer/classes/NodemailerEmailClient.ts`

**Configuration:**

| Env Variable | Description |
|-------------|-------------|
| `NODEMAILER_USER` | SMTP username (enables the service when set along with `NODEMAILER_PASS`) |
| `NODEMAILER_PASS` | SMTP password |
| `NODEMAILER_HOST` | SMTP host |
| `NODEMAILER_PORT` | SMTP port |

**Singleton:** `NodemailerClient` -- initialized at module load. Active when both `NODEMAILER_USER` and `NODEMAILER_PASS` are set.

**Transport configuration:**
- `secure: false`
- `tls.rejectUnauthorized: false` (noted as "To test in localhost")
- No connection pooling

| Method | Description |
|--------|-------------|
| `sendMessage(options)` | Send email with provided `SendMailOptions`. Throws `NodemailerEmailClientError` on failure. |
| `sendMessageFromLocalTemplate(options, templateName, variables)` | Inject HTML from a local template, then send via Nodemailer. |

### 3.3 Email Templates

**File:** `src/libs/emails/templates/`

All templates are functions `(vars: { [key: string]: string }) => string` that return inline-styled HTML strings.

| Template Key | File | Purpose | Variables |
|-------------|------|---------|-----------|
| `consentValidation` | `consentValidation.email.ts` | Consent grant confirmation email | `{ url }` -- link to confirm consent |
| `verification` | `verification.email.ts` | Account email verification | `{ url }` -- link to verify email |
| `test` | `test.email.ts` | Test email (renders all variables as list items) | Any key-value pairs |

**Template registry:**
```typescript
export const EmailTemplates = {
  test: testTemplate,
  verification: verificationTemplate,
  consentValidation: consentValidationTemplate,
};

export const EmailMailchimpTemplates = {};  // Empty -- no Mandrill templates configured
```

### 3.4 Email Usage in Consent Flows

Emails are sent during the consent grant flow when the user's identity needs to be verified via email. The `consentValidation` template is used to send a confirmation link containing all consent parameters as query string values, which the user clicks to complete the consent grant via the `GET /consents/emailverification` endpoint.

---

## 4. Session Management

**File:** `src/middleware/session.ts`

**Function:** `initSession()` -- returns an `express-session` middleware instance.

**Configuration:**

| Setting | Value | Description |
|---------|-------|-------------|
| Session ID | `crypto.randomUUID()` | UUID v4 generator |
| Secret | `process.env.SESSION_SECRET` | HMAC signing key |
| `resave` | `true` | Forces session to be saved on every request |
| `saveUninitialized` | `false` | Does not save empty sessions |
| Cookie name | `process.env.SESSION_COOKIE_NAME` | Configurable cookie name |
| `sameSite` | `"none"` (production) / `"lax"` (development) | Cross-site cookie policy |
| `secure` | `true` (production) / `false` (development) | HTTPS-only cookies in production |
| `maxAge` | 24 hours (86,400,000 ms) | Session lifetime |
| Store | MongoDB via `connect-mongo` | Persistent session storage |

**Store connection:** `MongoStore.create({ mongoUrl: process.env.MONGO_URI })`

**Session data structure:**

Sessions store authentication state for both users and participants:

```typescript
// After user login
req.session.user = {
  id: "<User ObjectId>",
  oath: {  // Note: misspelled key
    accessToken: "<JWT>",
    refreshToken: "<hex>"
  }
};

// After participant authentication
req.session.userParticipant = {
  id: "<Participant ObjectId>"
};
```

---

## 5. Logging

### 5.1 Winston Logger

**File:** `src/libs/loggers/Logger.ts`

The `Logger` class is a static wrapper around Winston with the following configuration:

**Log levels:**

| Level | Priority | Color |
|-------|----------|-------|
| `error` | 0 | Red |
| `warn` | 1 | Yellow |
| `info` | 2 | Green |
| `http` | 3 | Magenta |
| `debug` | 4 | Blue |

**Level selection:** `debug` in development, `http` in production.

**Transports:**

| Transport | Description | Configuration |
|-----------|-------------|---------------|
| Console | Colorized output to stdout | Always active |
| DailyRotateFile (error) | `logs/error/error_%DATE%.log` | Max files/size from env |
| DailyRotateFile (warn) | `logs/warn/warn_%DATE%.log` | Max files/size from env |
| DailyRotateFile (http) | `logs/http/http_%DATE%.log` | Max files/size from env |
| DailyRotateFile (info) | `logs/info/info_%DATE%.log` | Max files/size from env |
| DailyRotateFile (all) | `logs/all/all_%DATE%.log` | Hardcoded: 14 days, 20 MB |

**Environment variables:**

| Variable | Description |
|----------|-------------|
| `WINSTON_LOGS_MAX_FILES` | Maximum number of log files to retain (per level) |
| `WINSTON_LOGS_MAX_SIZE` | Maximum size per log file |

**Log format:** `YYYY-MM-DD HH:mm:ss <level>: <message> -- <location>`

**Static methods:**

| Method | Description |
|--------|-------------|
| `Logger.log(opts)` | General log with configurable level |
| `Logger.error(opts)` | Error-level log |
| `Logger.warn(opts)` | Warning-level log |
| `Logger.info(opts)` | Info-level log |
| `Logger.debug(opts)` | Debug-level log |
| `Logger.http(opts)` | HTTP-level log |
| `Logger.critical(opts)` | Critical log (placeholder: logs "SEND EMAIL TO ADMIN" to console) |
| `Logger.morganLog(message)` | HTTP request log from Morgan |

**Input formats:**

Methods accept either a `LoggerOptions` object or a plain string:
- `LoggerOptions`: `{ level?, message?, location?, callback? }`
- String: parsed by splitting on ` -- ` (location before separator, message after)

### 5.2 Morgan HTTP Logging

**File:** `src/libs/loggers/morganLogs.ts`

Morgan middleware configured with the `"combined"` format. Output is streamed to `Logger.morganLog`.

**Skip filter:** Requests to the following URL prefixes are not logged:
`web`, `favicon.ico`, `vt-admin`, `images`, `android`, `css`, `javascripts`, `stylesheets`, and root `/`.

---

## 6. Request Validation (Joi)

### 6.1 Validation Middleware

**File:** `src/middleware/joiValidation.ts`

**`setJoiValidationSchema` middleware:**
- Inspects `req.method` and `req.originalUrl`.
- If URL contains `"participants"`:
  - `POST` → `participantCreationSchema`
  - Other methods → `participantUpdateSchema`
- Stores selected schema on `req.validationSchema`.
- For non-participant routes, `schema` remains `undefined`.

**`validatePayload` middleware:**
- Calls `req.validationSchema.validate(req.body, { abortEarly: false, stripUnknown: true })`.
- `abortEarly: false` -- collects all validation errors.
- `stripUnknown: true` -- removes unrecognized fields from the payload.
- On validation error: returns `400` with `{ error: error.details }`.
- On success: replaces `req.body` with the validated and stripped payload.

### 6.2 Validation Schemas

**File:** `src/libs/joi/participantSchemas.ts`

**`participantCreationSchema`:**

| Field | Type | Required |
|-------|------|----------|
| `legalName` | string | yes |
| `email` | string | yes |
| `clientID` | string | yes |
| `clientSecret` | string | yes |
| `did` | string | no |
| `selfDescriptionURL` | string | no |
| `dataspaceEndpoint` | string | no |
| `endpoints.dataExport` | string | no |
| `endpoints.dataImport` | string | no |
| `endpoints.consentExport` | string | no |
| `endpoints.consentImport` | string | no |

**`participantUpdateSchema`:**

All fields optional: `legalName`, `selfDescriptionURL`, `email`, `endpoints.*`.

> **Note:** Only participant routes use Joi validation. No validation schemas exist for users, consents, or data exchange endpoints.

---

## 7. Error Handling

### 7.1 Global Error Handler

**File:** `src/middleware/globalErrorHandler.ts`

Express error middleware (4-argument signature). Mounted last in the middleware stack.

**Behavior:**

1. Log the error via `Logger.error({ message, location: stack })`.
2. Check the error type:
   - `BadRequestError` → `400` with `err.jsonResponse()`
   - `NotFoundError` → `404` with `{ error: err.message || "resource not found" }`
   - Everything else → `500` with `{ error: "Internal Server Error", message: "Something went wrong" }`
3. In development (`NODE_ENV === "development"`), the 500 response includes `devErr: { msg, stack, name }`.

### 7.2 Custom Error Classes

**Directory:** `src/errors/`

| Class | File | HTTP Status | Constructor | `jsonResponse()` Output |
|-------|------|-------------|-------------|------------------------|
| `BadRequestError` | `BadRequestError.ts` | 400 | `(message, fields: {field, message}[])` | `{ code: 400, error: message, errors: fields }` |
| `NotFoundError` | `NotFoundError.ts` | 404 | `(message?)` | `{ code: 404, error: "resource not found", message }` |
| `ForbiddenError` | `ForbiddenError.ts` | 403 | `(message)` | `{ code: 403, error: "forbidden action", message }` |
| `BadEndpointError` | `BadEndpointError.ts` | 404 | `(req: Request)` | `{ code: 404, error: "Invalid endpoint", message: "Invalid Endpoint. Path {path} is unknown" }` |
| `ThirdPartyError` | `ThirdPartyError.ts` | 400 | `(message?)` | `{ code: 400, error: "Third party service failed to provide results", message }` |

> **⚠ Bug:** The global error handler only explicitly handles `BadRequestError` and `NotFoundError`. All other custom errors (`ForbiddenError`, `ThirdPartyError`, `BadEndpointError`) fall through to the generic 500 handler despite having `jsonResponse()` methods and intended status codes. See [Known Bugs](#131-error-classes-not-handled).

### 7.3 ObjectId Format Validation

**File:** `src/middleware/objectIdFormatCheck.ts`

| Function | Type | Description |
|----------|------|-------------|
| `checkIDFormatMiddleware` | Middleware | Validates `req.params.id` as a MongoDB ObjectId. Throws `BadRequestError` on invalid format. |
| `isObjectID(id, fieldName?)` | Utility | Returns `true` if valid ObjectId, throws `BadRequestError` otherwise. Used directly in controller code. |

---

## 8. JSON-LD Self-Descriptions

**File:** `src/libs/jsonld/selfDescriptions.ts`

### 8.1 User Self-Description

**Function:** `userToSelfDescription(user: IUser)`

Generates a schema.org Person JSON-LD document:

```json
{
  "@context": "http://schema.org",
  "@type": "Person",
  "name": "<firstName> <lastName>",
  "email": "<email>",
  "url": "<URL>:<PORT if dev>/v1/users/<user.id>"
}
```

**Returns:** Pretty-printed JSON string (`JSON.stringify(jsonLd, null, 2)`)

**Called from:** `signup` controller (once, at user creation time)

> **Note:** The URL construction hardcodes `/v1/users/` instead of using `API_PREFIX`. If `API_PREFIX` is set to something other than `/v1`, the self-description URL will be wrong.

### 8.2 Participant Self-Description

There is no equivalent self-description generator for participants. During participant registration, `jsonld` is hardcoded to `""`. The participant's JSON-LD is expected to be provided externally.

---

## 9. Simulated Contract Service

**File:** `src/simulated/contract/router.ts`

A development-mode mock of the Contract Service, mounted at `/contracts` in `src/server.ts`. It provides hardcoded ODRL Policy contract data for testing without an external Contract Service.

### 9.1 Routes

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/contracts/` | Returns all simulated contracts (array of 2) |
| `GET` | `/contracts/contracts/:providerID/:consumerID` | Filtered contracts by provider/consumer |
| `GET` | `/contracts/validate/:providerID/:consumerID` | Always returns `{ success: true }` |

### 9.2 Contract Data

The simulated service provides two hardcoded ODRL Policy contracts with:
- Unique `uid` identifiers
- Permission blocks with `assigner` (provider) and `assignee` (consumer) URLs
- Data type references
- Purpose definitions with consent types and PII categories
- Temporal and spatial constraints

The assigner/assignee URLs use `process.env.FEDERATED_APPLICATION_IDENTIFIER` as a prefix with hardcoded ObjectIds.

### 9.3 Filtering Logic

The `GET /contracts/contracts/:providerID/:consumerID` endpoint:
1. URL-decodes both `providerID` and `consumerID` parameters.
2. Filters contracts where any permission's `assignee` matches either the provider or consumer ID.

> **⚠ Bug:** The filtering checks `assignee` for both parties instead of checking `assigner` for the provider and `assignee` for the consumer. This would return incorrect results in contracts where the provider and consumer have different roles.

---

## 10. API Documentation (Swagger)

**Mounting point:** `/docs` (no API prefix)

**File:** `docs/swagger.json` (OpenAPI 3.0 specification)

**Setup** (`src/routes/index.ts`):
```typescript
app.use("/docs", swaggerUI.serve, swaggerUI.setup(swaggerSpec));
```

The Swagger UI is served at the `/docs` path and renders the OpenAPI specification from `docs/swagger.json`. This provides interactive API documentation for all endpoints.

> **Note:** The Swagger documentation is mounted at `/docs` without the `API_PREFIX`, so it is always accessible at the same path regardless of API versioning configuration.

---

## 11. Health Check

**Endpoint:** `GET /health` `[None]`

**Mounting point:** App root (no API prefix)

**Response:** `200`
```json
{ "status": "OK" }
```

**Implementation:** Inline handler in `src/routes/index.ts`:
```typescript
app.get("/health", (req, res) => {
  res.json({ status: "OK" });
});
```

**Characteristics:**
- No authentication required
- No database connectivity check
- No dependency health checks
- Functions as a simple liveness probe (is the process running and accepting HTTP connections?)

---

## 12. Server Setup and Middleware Stack

**File:** `src/server.ts`

### 12.1 Initialization

**Function:** `startServer(testPort?, agentConfigPath?)` (async)

1. If no `testPort`, connect to MongoDB via `loadMongoose()`.
2. Create Express app.
3. Apply middleware stack (see below).
4. Initialize Consent Agent (if config file exists).
5. Load routes.
6. Mount simulated contract service.
7. Start HTTP listener.
8. Return `{ server, app }` for test access.

### 12.2 Middleware Stack (Order)

| Order | Middleware | Description |
|-------|-----------|-------------|
| 1 | EJS view engine | Template engine for consent.ejs and email templates |
| 2 | CORS | `origin: true, credentials: true` -- allows all origins with cookies |
| 3 | `express.json()` | JSON body parser |
| 4 | `trust proxy` | Enables proxy-aware IP detection |
| 5 | `initSession()` | Express-session with MongoDB store |
| 6 | `morganLogs` | HTTP request logging via Morgan → Winston |
| 7 | Route handlers | All API routes |
| 8 | Simulated contract router | Development mock at `/contracts` |
| 9 | `globalErrorHandler` | Express error handler (mounted last via `loadRoutes`) |

### 12.3 Route Mounting

**File:** `src/routes/index.ts`

| Path | Handler | Auth |
|------|---------|------|
| `GET /health` | Inline health check | None |
| `{API_PREFIX}/consents` | `consentRouter` | Per-route |
| `{API_PREFIX}/users` | `usersRouter` | Per-route |
| `{API_PREFIX}/participants` | `participantsRouter` | Per-route |
| `{API_PREFIX}/data-exchange` | `dataExchangeRouter` | Per-route |
| `{API_PREFIX}/` | `ConsentAgentRouter` | UserJWT |
| `/docs` | Swagger UI | None |
| *(last)* | `globalErrorHandler` | N/A |

### 12.4 EJS Views

**Directory:** `src/views/`

Contains `consent.ejs` -- used by the dead OAuth `/authorize` route to render a consent approval page.

---

## 13. Known Bugs

### 13.1 Error Classes Not Handled

**Location:** `src/middleware/globalErrorHandler.ts`

**Description:** The global error handler only explicitly handles `BadRequestError` (→ 400) and `NotFoundError` (→ 404). Three other custom error classes -- `ForbiddenError` (intended 403), `ThirdPartyError` (intended 400), and `BadEndpointError` (intended 404) -- all fall through to the generic 500 handler despite having `jsonResponse()` methods with correct status codes.

**Recommendation:** Add explicit handlers for all custom error classes, or use a common base class with a `statusCode` property.

### 13.2 Nodemailer Error Logging Misattributed

**Location:** `src/libs/emails/nodemailer/classes/NodemailerEmailClient.ts`

**Description:** The error logging location strings reference `"MailchimpEmailClient.sendMessage"` and `"MailchimpEmailClient.sendMessageFromLocalTemplate"` instead of `"NodemailerEmailClient.*"`. This is a copy-paste bug from the Mailchimp implementation.

### 13.3 `generateNewAccessToken` Is an Exact Duplicate

**Location:** `src/libs/OAuth/tokens.ts`

**Description:** The `generateNewAccessToken` function is a character-for-character duplicate of `generateAccessToken`. Both create JWTs with the same payload, same secret, and same expiry. One should be removed.

### 13.4 Simulated Contract Filtering Bug

**Location:** `src/simulated/contract/router.ts`

**Description:** The `GET /contracts/:providerID/:consumerID` endpoint filters contracts by checking if either the provider or consumer matches a permission's `assignee`. It should check `assigner` for the provider and `assignee` for the consumer.

### 13.5 User JSON-LD URL Hardcodes API Prefix

**Location:** `src/libs/jsonld/selfDescriptions.ts`

**Description:** The `url` field in the user self-description hardcodes `/v1/users/` instead of reading from `process.env.API_PREFIX`. If the API prefix is changed, the self-description URL will be incorrect.

### 13.6 TLS Disabled in Nodemailer

**Location:** `src/libs/emails/nodemailer/classes/NodemailerEmailClient.ts`

**Description:** The Nodemailer transport is configured with `tls: { rejectUnauthorized: false }`, which disables certificate verification. This is marked "To test in localhost" but has no production override. In production, this allows man-in-the-middle attacks on email sending.

**Recommendation:** Make TLS verification configurable via an environment variable, defaulting to `true` in production.

### 13.7 Critical Log Does Not Actually Send Email

**Location:** `src/libs/loggers/Logger.ts` (`critical` method)

**Description:** The `critical` method has a callback that logs `"SEND EMAIL TO ADMIN -- CRITICAL LOG REQUESTED"` to `console.error`, but does not actually send any email. The admin notification is a placeholder.

### 13.8 Empty Mailchimp Templates Registry

**Location:** `src/libs/emails/templates/index.ts`

**Description:** `EmailMailchimpTemplates` is exported as an empty object `{}`. The `MailchimpEmailClient.sendMessageFromTemplate` method references `(EmailMailchimpTemplates as any)[templateName].slug`, which would throw a runtime error if called because no templates are registered.
