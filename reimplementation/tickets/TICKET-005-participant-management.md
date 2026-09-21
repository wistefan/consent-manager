# TICKET-005: Participant Management

## Metadata

| Field | Value |
|-------|-------|
| **Ticket ID** | TICKET-005 |
| **Title** | Participant Management |
| **Priority** | Critical |
| **Complexity** | M (Medium) |
| **Dependencies** | TICKET-002, TICKET-003 |
| **Related User Stories** | US-PM-001, US-PM-002, US-PM-003, US-PM-004, US-PM-005, US-PM-006, US-PM-007, US-PM-008, US-PM-009 |

## Description

Implement participant (data provider/consumer) registration, login, CRUD operations, DSC integration, and public key management. Participants are organizational entities that register with the Consent Manager to participate in data exchanges.

## Technical Details

### Endpoints

#### 1. Register Participant — `POST /participants`

**Auth:** None (public registration)

**Request Body:** Validated by Joi schema

```json
{
  "legalName": "string (required)",
  "did": "string (optional)",
  "selfDescriptionURL": "string (optional)",
  "email": "string (optional)",
  "legalPerson": {
    "registrationNumber": "string",
    "headquartersAddress": "string",
    "legalAddress": "string",
    "parentOrganization": "string",
    "subOrganization": "string"
  },
  "clientID": "string (required)",
  "clientSecret": "string (required)",
  "dataspaceEndpoint": "string (optional)",
  "endpoints": {
    "dataExport": "string",
    "dataImport": "string",
    "consentExport": "string",
    "consentImport": "string"
  }
}
```

**Flow:**
1. Validate request body with Joi
2. Check for duplicate `clientID`
3. Hash `clientSecret` with bcrypt before storage
4. Create Participant document
5. Generate JSON-LD self-description
6. If `dataspaceEndpoint` is provided, perform DSC integration:
   a. POST login to DSC with `clientID`/`clientSecret`
   b. POST consent manager's public key to DSC
   c. POST consent manager's URI to DSC
7. Return created participant (excluding secrets)

**Response:** `201 Created`

#### 2. Participant Login — `POST /participants/login`

**Auth:** None (public)

**Request Body:**
```json
{
  "clientID": "string (required)",
  "clientSecret": "string (required)"
}
```

**Flow:**
1. Find participant by `clientID`
2. Verify `clientSecret` against stored hash
3. Generate JWT with 1-hour expiry, `sub` = participant ID
4. Create session with `req.session.participant`
5. Return JWT

**Response:** `200 OK` with `{ token: "jwt_string" }`

#### 3. List All Participants — `GET /participants`

**Auth:** None (public, paginated)

**Query Parameters:** `page` (default: 1), `limit` (default: 20)

**Flow:**
1. Query participants with pagination
2. Exclude `clientID` and `clientSecret` from results
3. Return paginated list

**Response:** `200 OK` with `{ participants: [...], total: number, page: number, limit: number }`

#### 4. Get Participant by ID — `GET /participants/:id`

**Auth:** None (public)

**Flow:**
1. Validate ObjectId format
2. Find participant by ID, excluding `clientID` and `clientSecret`
3. Return participant

**Response:** `200 OK` with participant object

#### 5. Get Participant by Client ID — `GET /participants/clientId/:clientId`

**Auth:** None (public)

**Flow:**
1. Find participant by `clientID`, excluding secrets
2. Return participant

**Response:** `200 OK` with participant object

#### 6. View My Participant Profile — `GET /participants/me`

**Auth:** `verifyParticipantJWT`

**Flow:**
1. Read authenticated participant from `req.participant`
2. Return participant data, excluding `clientSecret`

**Response:** `200 OK` with participant object

#### 7. Update Participant Endpoints — `PUT /participants/clientId/:clientId`

**Auth:** `verifyParticipantJWT`

**Request Body:** Validated by Joi update schema

```json
{
  "endpoints": {
    "dataExport": "string",
    "dataImport": "string",
    "consentExport": "string",
    "consentImport": "string"
  },
  "dataspaceEndpoint": "string"
}
```

**Flow:**
1. Validate request body with Joi
2. Find participant by `clientId`
3. Update fields
4. Save and return updated participant (with `{ new: true }`)

**Response:** `200 OK` with updated participant

#### 8. Delete My Participant — `DELETE /participants/me`

**Auth:** `verifyParticipantJWT`

**Flow:**
1. Delete the authenticated participant
2. Clean up associated UserIdentifiers (fixing original cascade bug)
3. Destroy session

**Response:** `200 OK`

#### 9. Get Consent Signature Public Key — `GET /participants/consent-signature`

**Auth:** None (public)

**Flow:**
1. Read the RSA public key from key storage
2. Return the PEM-encoded public key

**Response:** `200 OK` with `{ publicKey: "pem_string" }`

### Joi Validation Schemas

**File:** `src/libs/joi/participantSchemas.ts`

Define Joi schemas for:
- `participantCreateSchema` — Validates registration body
- `participantUpdateSchema` — Validates update body (subset of fields)

### Field Selection Constant

```typescript
const PARTICIPANT_PUBLIC_FIELDS = "-clientID -clientSecret";
```

### DSC Integration Flow

When a participant provides a `dataspaceEndpoint`:
1. **Login:** `POST {dataspaceEndpoint}/login` with `{ clientID, clientSecret }`
2. **Push Public Key:** `POST {dataspaceEndpoint}/consent/public-key` with the consent signature public key
3. **Push URI:** `POST {dataspaceEndpoint}/consent/uri` with the Consent Manager's URL

Handle DSC integration failures gracefully — log the error but still save the participant.

### Files to Create/Modify

```
src/controllers/participantsController.ts  # Controller functions
src/routes/participants.ts                 # Route definitions
src/routes/index.ts                        # Mount participants router
src/libs/joi/participantSchemas.ts         # Joi validation schemas
```

## Acceptance Criteria

1. [ ] `POST /participants` creates a participant with hashed `clientSecret`
2. [ ] `POST /participants` rejects duplicate `clientID` with `409 Conflict`
3. [ ] `POST /participants` performs DSC integration when `dataspaceEndpoint` is provided
4. [ ] `POST /participants/login` authenticates and returns a 1-hour JWT
5. [ ] `GET /participants` returns paginated list without secrets
6. [ ] `GET /participants/:id` returns participant without secrets
7. [ ] `GET /participants/me` returns authenticated participant without `clientSecret`
8. [ ] `PUT /participants/clientId/:clientId` updates endpoints and returns updated document
9. [ ] `DELETE /participants/me` deletes participant and cleans up UserIdentifiers
10. [ ] `GET /participants/consent-signature` returns the RSA public key
11. [ ] Joi validation rejects invalid registration/update payloads
12. [ ] Integration tests cover registration, login, CRUD, and DSC integration

## Known Issues to Address

1. **`getMyParticipant` exposes secrets** (original bug 12.1): Always use field selection to exclude `clientID` and `clientSecret` from responses (except where needed for auth).
2. **Update `save()` not awaited** (original bug 12.2): Always `await` all database write operations.
3. **`clientSecret` stored in plaintext** (original bug 12.3): Hash with bcrypt before storage.
4. **Registration `jsonld` dead code** (original bug 12.4): Generate JSON-LD properly or skip if not needed.
5. **DSC integration errors not handled** (original bug 12.5): Catch DSC errors gracefully — save the participant regardless, log the DSC failure as a warning.
6. **No cascade on deletion** (original bug 12.6): When deleting a participant, remove associated UserIdentifiers and consider handling orphaned consents.
7. **`validatePassword` declared but not implemented** (original bug 12.7): If `clientSecret` is hashed, implement a proper validation method on the Participant model.
