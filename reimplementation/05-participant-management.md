# Participant Management

This document provides comprehensive feature documentation for the participant management module. It covers participant registration with DSC integration, authentication (login), CRUD operations, public key management, and the configuration exchange between the Consent Manager and participant Data Space Connectors (DSCs).

> **Source files:** `src/controllers/participantsController.ts`, `src/routes/participants.ts`, `src/models/Participant/Participant.model.ts`, `src/middleware/auth.ts`, `src/middleware/joiValidation.ts`, `src/libs/joi/participantSchemas.ts`, `src/libs/jwt/index.ts`

---

## Table of Contents

1. [Participant Registration](#1-participant-registration)
2. [Participant Login](#2-participant-login)
3. [Participant Retrieval](#3-participant-retrieval)
4. [Participant Update](#4-participant-update)
5. [Participant Deletion](#5-participant-deletion)
6. [Public Key Management](#6-public-key-management)
7. [DSC Integration](#7-dsc-integration)
8. [Joi Validation](#8-joi-validation)
9. [Authentication: `verifyParticipantJWT`](#9-authentication-verifyparticipantjwt)
10. [Field Selection](#10-field-selection)
11. [Endpoint Reference](#11-endpoint-reference)
12. [Known Bugs](#12-known-bugs)

---

## 1. Participant Registration

**Endpoint:** `POST /participants` `[None]` *(Joi-validated)*

**Middleware chain:** `setJoiValidationSchema` → `validatePayload` → `registerParticipant`

**Request body** (validated by `participantCreationSchema`):

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `legalName` | string | yes | Legal name of the participant organization |
| `email` | string | yes | Contact email |
| `clientID` | string | yes | Unique service key used for identification |
| `clientSecret` | string | yes | Shared secret for JWT authentication |
| `did` | string | no | Decentralized Identifier |
| `selfDescriptionURL` | string | no | URL pointing to the participant's self-description |
| `dataspaceEndpoint` | string | no | URL of the participant's Data Space Connector |
| `endpoints.dataExport` | string | no | DSC data export endpoint |
| `endpoints.dataImport` | string | no | DSC data import endpoint |
| `endpoints.consentExport` | string | no | DSC consent export endpoint |
| `endpoints.consentImport` | string | no | DSC consent import endpoint |

**Logic flow:**

1. Joi validation strips unknown fields and validates all required fields.
2. Check for existing participant with the same `clientID` via `Participant.findOne({ clientID })`.
3. If duplicate found, return `409`.
4. Create a new `Participant` document from the validated body.
5. Force `jsonld = ""` (hardcoded; self-description generation is not implemented for participants).
6. Set `selfDescriptionURL` from body or default to `""`.
7. Dead code block: if `jsonld` were non-empty, parse it and extract endpoint URLs. This never executes because `jsonld` is always `""`.
8. Set `clientID` and `clientSecret` from body.
9. **If `dataspaceEndpoint` is provided**, perform DSC integration (see [Section 7](#7-dsc-integration)):
   - Log in to the participant's DSC.
   - Push the Consent Manager's public key and URI to the DSC.
10. Save the participant.

**Response:** `201` -- Full participant document (including `clientID` and `clientSecret`)

**Errors:**
- `400` -- Joi validation errors (`{ error: [details] }`)
- `409` -- `{ error: "Participant already exists" }`

> **Note:** Registration is public (no authentication required). The source code comment suggests this should be restricted to catalog instances: "Should there be extra api key in here to only allow catalog instances?"

---

## 2. Participant Login

**Endpoint:** `POST /participants/login` `[None]`

**Request body:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `clientID` | string | yes | Participant's service key |
| `clientSecret` | string | yes | Participant's shared secret |

**Logic flow:**

1. Query `Participant.findOne({ clientID, clientSecret })` -- plaintext credential match (no hashing).
2. If not found, return `404`.
3. Generate a JWT token using `issueJwt(participant)`:
   - Payload: `{ sub: participant.id, participant_name: participant.legalName }`
   - Signed with `process.env.JWT_SECRET_KEY`
   - Expires in 1 hour

**Response:** `200`
```json
{
  "success": true,
  "jwt": "<JWT token string>",
  "message": "Successfully retrieved JWT for authenticating requests to the API"
}
```

**Errors:**
- `404` -- `{ error: "participant not found" }`

> **Note:** The `clientSecret` is stored and compared in plaintext. Unlike user passwords (bcrypt-hashed), participant credentials have no cryptographic protection in the database.

---

## 3. Participant Retrieval

### 3.1 Get All Participants (Paginated)

**Endpoint:** `GET /participants` `[None]`

**Query parameters:**

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `limit` | string | `"10"` | Number of results per page |
| `page` | string | `"1"` | Page number (1-indexed) |

**Logic flow:**

1. Compute `skip = (page - 1) * limit` (floor at 0).
2. Query `Participant.find()` with `.skip(skip).limit(limit).select(PARTICIPANT_SELECTION)`.
3. Count all participants with `Participant.countDocuments()`.
4. Compute `totalPages = ceil(totalCount / limit)`.

**Response:** `200`
```json
{
  "participants": [ /* sanitized participant objects */ ],
  "totalCount": 42,
  "totalPages": 5
}
```

### 3.2 Get Participant by ID

**Endpoint:** `GET /participants/:id` `[None]`

**Parameters:** `id` -- MongoDB ObjectId

**Response:** `200` -- Sanitized participant object (without `clientID`, `clientSecret`, `password`)

**Errors:**
- `404` -- `{ error: "participant not found" }`

### 3.3 Get Participant by Client ID

**Endpoint:** `GET /participants/clientId/:clientId` `[None]`

**Parameters:** `clientId` -- The participant's service key

**Logic:** Queries `Participant.findOne({ clientID: clientId }).select(PARTICIPANT_SELECTION)`.

**Response:** `200` -- Sanitized participant object

**Errors:**
- `404` -- `{ error: "participant not found" }`

### 3.4 Get My Participant (Authenticated)

**Endpoint:** `GET /participants/me` `[ParticipantJWT]`

**Logic:** Queries `Participant.findById(req.userParticipant?.id)` with **no field selection** -- returns the full document including secrets.

**Response:** `200` -- Full participant document (includes `clientID` and `clientSecret`)

**Errors:**
- `NotFoundError("Participant not found")` (handled by global error handler → 404)

> **⚠ Bug:** Unlike the public retrieval endpoints, this endpoint does not apply `PARTICIPANT_SELECTION`, so it exposes `clientID` and `clientSecret`. See [Known Bugs](#121-getmyparticipant-exposes-secrets).

---

## 4. Participant Update

**Endpoint:** `PUT /participants/clientId/:clientId` `[ParticipantJWT]`

**Parameters:** `clientId` -- The participant's service key

**Request body:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `dataExport` | string | no | Updated data export endpoint |
| `dataImport` | string | no | Updated data import endpoint |
| `consentExport` | string | no | Updated consent export endpoint |
| `consentImport` | string | no | Updated consent import endpoint |
| `dataspaceEndpoint` | string | no | Updated Data Space Connector endpoint |

**Logic flow:**

1. Find participant by `clientID` with `PARTICIPANT_SELECTION`.
2. For each provided field, update the participant's corresponding endpoint. If a field is absent or falsy, keep the existing value.
3. Call `participant.save()` (without `await`).

**Response:** `200` -- Participant object (sanitized)

**Errors:**
- `404` -- `{ error: "participant not found" }`

> **⚠ Bug:** The `participant.save()` call is not awaited. The response is sent before the database write completes. See [Known Bugs](#122-update-save-not-awaited).

> **Note:** Despite Joi validation schemas existing for participant updates (`participantUpdateSchema`), this route does **not** use the Joi validation middleware chain. Only the POST `/` route uses Joi.

---

## 5. Participant Deletion

**Endpoint:** `DELETE /participants/me` `[ParticipantJWT]`

**Logic flow:**

1. Get the authenticated participant's ID from `req.userParticipant.id`.
2. Call `Participant.findByIdAndDelete(id)`.
3. If not found, throw `NotFoundError`.

**Response:** `200`
```json
{ "message": "Participant deleted successfully" }
```

**Errors:**
- `NotFoundError("Participant not found")` → 404

> **Note:** Deleting a participant does not cascade to UserIdentifiers, Consents, or PrivacyNotices associated with that participant. Orphaned references remain.

---

## 6. Public Key Management

**Endpoint:** `GET /participants/consent-signature` `[ParticipantJWT]`

**Logic flow:**

1. Read the file `src/config/keys/consentSignaturePublic.pem` synchronously as base64.
2. Return the base64-encoded public key.

**Response:** `200`
```json
{ "key": "<base64-encoded PEM public key>" }
```

**Purpose:** Allows participants to retrieve the Consent Manager's RSA public key for verifying consent signatures. The corresponding private key is used by the Consent Manager to sign (encrypt) consent payloads during data exchange.

### Deprecated: Export Public Key to All Participants

**Function:** `exportPublicKeyToParticipants` (defined but route commented out)

This function iterates over all participants, logs in to each DSC, and pushes the public key and consent manager URI. It is deprecated and no longer accessible via any route.

---

## 7. DSC Integration

When a participant provides a `dataspaceEndpoint` during registration, the Consent Manager performs a three-step handshake with the participant's Data Space Connector:

### 7.1 Registration-Time DSC Configuration

**Trigger:** `dataspaceEndpoint` field present in `POST /participants` request body.

**Step-by-step flow:**

1. **Fetch DSC self-description:** `GET <dataspaceEndpoint>`
   - Expected response structure: `{ content: { _links: { login: { href }, consentConfiguration: { href } } } }`
   - The response uses a HATEOAS-style link structure.

2. **Read consent public key:** Load `src/config/keys/consentSignaturePublic.pem` as base64.

3. **Authenticate with the DSC:** `POST <_links.login.href>` with:
   ```json
   { "serviceKey": "<clientID>", "secretKey": "<clientSecret>" }
   ```
   - Expected response: `{ content: { token: "<JWT>" } }`

4. **Push consent configuration:** `PUT <_links.consentConfiguration.href>` with:
   ```json
   {
     "publicKey": "<base64 PEM public key>",
     "uri": "<consent manager base URL with API prefix>"
   }
   ```
   - Authorization: `Bearer <token from step 3>`
   - The URI is constructed as:
     - Development: `${URL}:${PORT}${API_PREFIX}/`
     - Production: `${URL}${API_PREFIX}/`

### 7.2 DSC Endpoints Called During Consent Flows

Beyond registration, the Consent Manager also calls participant DSC endpoints during consent granting (in `consentsController.ts`):

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `<dataspaceEndpoint>/login` | POST | Authenticate before user operations |
| `<dataspaceEndpoint>/private/users/register` | POST | Register a UserIdentifier on the DSC |
| `<dataspaceEndpoint>/private/users/app` | POST | Register a user for a specific consent/app |
| `<endpoints.consentExport>` | POST | Trigger data exchange (encrypted consent payload) |
| `<endpoints.consentImport>` | POST | Forward consent to consumer DSC |

### 7.3 Trust Model

The public key exchange during registration establishes the trust relationship:
- The Consent Manager shares its **RSA public key** so the DSC can verify consent signatures.
- The Consent Manager shares its **base URI** so the DSC knows where to call back (e.g., `attachTokenToConsent`).
- The DSC shares its endpoints (`consentExport`, `consentImport`, `dataExport`, `dataImport`) so the Consent Manager knows how to trigger data exchanges.

---

## 8. Joi Validation

### 8.1 Validation Middleware

**Middleware:** `setJoiValidationSchema` → `validatePayload` (`src/middleware/joiValidation.ts`)

**`setJoiValidationSchema`:**
- Inspects `req.method` and `req.originalUrl`.
- If URL contains `"participants"`:
  - `POST` → assigns `participantCreationSchema`
  - Any other method → assigns `participantUpdateSchema`
- Stores the schema on `req.validationSchema`.

**`validatePayload`:**
- Calls `req.validationSchema.validate(req.body, { abortEarly: false, stripUnknown: true })`.
- `abortEarly: false` -- collects all validation errors, not just the first.
- `stripUnknown: true` -- removes any fields not defined in the schema.
- On error: returns `400` with `{ error: error.details }`.
- On success: replaces `req.body` with the validated and stripped payload.

### 8.2 Validation Schemas

**`participantCreationSchema`** (`src/libs/joi/participantSchemas.ts`):

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

**`participantUpdateSchema`**:

| Field | Type | Required |
|-------|------|----------|
| `legalName` | string | no |
| `selfDescriptionURL` | string | no |
| `email` | string | no |
| `endpoints.*` | string | no |

> **Note:** Only the `POST /participants` route uses Joi validation. The `PUT /participants/clientId/:clientId` route does not use the validation middleware despite the `participantUpdateSchema` being defined.

---

## 9. Authentication: `verifyParticipantJWT`

**Location:** `src/middleware/auth.ts`

The `verifyParticipantJWT` middleware supports two distinct authentication paths:

### Path 1: Session-Based (Cached)

If `req.session.userParticipant` exists (set during a previous request), the middleware sets `req.userParticipant = { id: session.userParticipant.id }` and proceeds immediately.

### Path 2: Bearer Token

Falls back to the `Authorization: Bearer <token>` header. If no header is present but `req.query.participant` exists, the query parameter is synthesized into the Authorization header.

The token is decoded (base64 of the JWT payload segment) to check for a `serviceKey` field:

**Sub-path A: Self-Issued JWT (no `serviceKey`):**
- Verifies the token against `process.env.JWT_SECRET_KEY`.
- Extracts `sub` claim as the participant's ObjectId.
- Stores in session for future requests.

**Sub-path B: External/Catalog JWT (has `serviceKey`):**
- Looks up the Participant by `clientID === serviceKey`.
- Verifies the token against that participant's `clientSecret` (used as the JWT signing key).
- Stores in session for future requests.

**Error responses:**
- `401` -- `{ message: "Authorization header missing or invalid" }`
- `401` -- `{ message: "'<token>' is not a valid authorization token" }` (less than 3 JWT segments)
- `401` -- `{ message: "Invalid or expired token" }` (JWT verification failed)
- `401` -- `{ success: false, message: "Unauthorized resource" }` (participant not found by serviceKey)

---

## 10. Field Selection

**Constant:** `PARTICIPANT_SELECTION = "-password -clientID -clientSecret"` (`src/utils/schemaSelection.ts`)

Applied to all public retrieval endpoints (`GET /`, `GET /:id`, `GET /clientId/:clientId`) to exclude sensitive fields from the response. The Participant model does not actually have a `password` field (that exclusion is a no-op), but `clientID` and `clientSecret` are correctly excluded.

**Exception:** `GET /participants/me` does **not** apply this selection, exposing all fields to the authenticated participant.

---

## 11. Endpoint Reference

| # | Method | Path | Auth | Middleware | Controller | Description |
|---|--------|------|------|------------|------------|-------------|
| 1 | `GET` | `/participants` | None | -- | `getAllParticipants` | Paginated list of all participants |
| 2 | `GET` | `/participants/me` | ParticipantJWT | -- | `getMyParticipant` | Get authenticated participant (full document) |
| 3 | `GET` | `/participants/clientId/:clientId` | None | -- | `getParticipantByClientId` | Find participant by service key |
| 4 | `GET` | `/participants/consent-signature` | ParticipantJWT | -- | `getPublicKey` | Get consent signing public key |
| 5 | `GET` | `/participants/:id` | None | -- | `getParticipantById` | Get participant by MongoDB ID |
| 6 | `PUT` | `/participants/clientId/:clientId` | ParticipantJWT | -- | `updateParticipantByClientId` | Update endpoints and dataspace URL |
| 7 | `POST` | `/participants` | None | Joi validation | `registerParticipant` | Register new participant |
| 8 | `POST` | `/participants/login` | None | -- | `loginParticipant` | Authenticate and get JWT |
| 9 | `DELETE` | `/participants/me` | ParticipantJWT | -- | `deleteParticipant` | Delete authenticated participant |

---

## 12. Known Bugs

### 12.1 `getMyParticipant` Exposes Secrets

**Location:** `src/controllers/participantsController.ts` (`getMyParticipant` function)

**Description:** The `GET /participants/me` endpoint queries `Participant.findById(id)` without `PARTICIPANT_SELECTION`, so the response includes `clientID` and `clientSecret` in plaintext. While this may be intentional (the authenticated participant viewing its own credentials), it is inconsistent with all other retrieval endpoints.

### 12.2 Update `save()` Not Awaited

**Location:** `src/controllers/participantsController.ts` (`updateParticipantByClientId` function)

**Description:** The `participant.save()` call on the last line of the function is not `await`ed. The HTTP response is sent before the database write completes. If the save fails, the error is silently swallowed and the client receives a success response.

### 12.3 Participant `clientSecret` Stored in Plaintext

**Location:** `src/models/Participant/Participant.model.ts`

**Description:** The `clientSecret` is stored as a plain string without hashing. Unlike the `User.password` field (which uses a bcrypt pre-save hook), participant credentials have no cryptographic protection in the database.

**Impact:** If the database is compromised, all participant secrets are immediately exposed.

**Recommendation:** Apply bcrypt hashing to `clientSecret` and compare using a `validateSecret` instance method (similar to `User.validatePassword`).

### 12.4 Registration `jsonld` Dead Code

**Location:** `src/controllers/participantsController.ts` (`registerParticipant` function)

**Description:** The registration flow sets `newParticipant.jsonld = ""` (hardcoded), then has a conditional block `if (newParticipant.jsonld !== "")` that parses the JSON-LD and extracts endpoints. This conditional never executes because `jsonld` was just set to an empty string.

### 12.5 DSC Integration Errors Not Handled Gracefully

**Location:** `src/controllers/participantsController.ts` (`registerParticipant` function)

**Description:** If the DSC integration fails (steps 3-4 in [Section 7.1](#71-registration-time-dsc-configuration)), the error propagates to the global error handler, which returns a generic 500 response. The participant is not saved. A more graceful approach would save the participant and flag the DSC integration as failed for later retry.

### 12.6 No Cascade on Participant Deletion

**Location:** `src/controllers/participantsController.ts` (`deleteParticipant` function)

**Description:** Deleting a participant does not clean up associated UserIdentifiers (whose `attachedParticipant` references the deleted participant), PrivacyNotices, or Consents. Orphaned references remain throughout the system.

### 12.7 `IParticipantModel` Declares `validatePassword` Method That Does Not Exist

**Location:** `src/types/models.ts`

**Description:** The TypeScript interface `IParticipantModel` declares a `validatePassword` method, but the Participant Mongoose schema has no such method defined. This is a dead declaration.
