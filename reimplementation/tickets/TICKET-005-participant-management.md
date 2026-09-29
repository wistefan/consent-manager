# TICKET-005: Participant Management

## Metadata

| Field | Value |
|-------|-------|
| **Ticket ID** | TICKET-005 |
| **Title** | Participant Management |
| **Priority** | Critical |
| **Complexity** | M (Medium) |
| **Dependencies** | TICKET-002, TICKET-003 |
| **Related User Stories** | US-PM-001, US-PM-002, US-PM-003, US-PM-004, US-PM-005, US-PM-006, US-PM-007, US-PM-008 |

## Description

Implement participant registration and management. A participant is an organization acting as data
provider or data consumer.

Two principles shape this ticket:

1. **Participants authenticate through the external IDP.** The Consent Manager stores no client ID,
   no client secret, and no credential of any kind, and offers no login endpoint.
2. **Participants carry their data space identity; they do not operate it here.** A participant's
   global identifier, self-description and data space membership belong to the data space. The
   Consent Manager records the identifier and treats everything attached to it as opaque.

## Technical Details

### Identity and Onboarding

A participant record is created **self-service, from the token**. Any token bearing the
`PARTICIPANT` role may create and maintain its own record. The participant identifier is always read
from the configured claim — never from the request body, a path parameter, or a query string. The
IDP is the gatekeeper; there is no separate approval workflow and no `ADMIN` role.

The identifier is the data space global participant ID: a DID, a URI, or an IDP-issued string. The
Consent Manager does not parse it, resolve it, or attach meaning to its form.

### Endpoints

#### 1. Register My Participant — `POST /participants` (US-PM-001)

**Auth:** `PARTICIPANT`

**Request Body:**
```json
{
  "legalName": "string (required)",
  "selfDescriptionUri": "string (optional)",
  "email": "string (optional)",
  "legalPerson": {
    "registrationNumber": "string (optional)",
    "headquartersAddress": "string (optional)",
    "legalAddress": "string (optional)",
    "parentOrganization": "string (optional)",
    "subOrganization": "string (optional)"
  },
  "endpoints": {
    "consentNotification": "string (optional — https URL)"
  }
}
```

**Flow:**
1. Read the participant identifier from the token.
2. If a record already exists for that identifier → **409 Conflict** (use `PUT /participants/me`).
3. Validate the body with Bean Validation.
4. Persist the record.

**Response:** `201 Created` with the participant representation.

Notably absent: `clientID`, `clientSecret`, `dataspaceEndpoint`, and the four DSC endpoints
(`dataExport`, `dataImport`, `consentExport`, `consentImport`). `selfDescriptionUri` is stored as an
opaque string — it is **not** fetched, resolved, or validated against anything.

`endpoints.consentNotification` is the one endpoint that survives, and it is a consent concern, not
a data-exchange one: it is where TICKET-020 delivers consent state changes so that a revocation
actually reaches the participants bound by it.

#### 2. Authenticate as a Participant (US-PM-002)

**There is no login endpoint.** A participant obtains a token from a registered IDP using whatever
grant that IDP offers — typically `client_credentials` — and presents it as
`Authorization: Bearer <token>`.

The work item here is **documentation, not code**: the token contract (TICKET-003 §3) is published
in the API documentation together with a worked example of obtaining and using a participant token.
The validation behaviour itself is owned by TICKET-003 and covered by its acceptance criteria.

#### 3. List Participants — `GET /participants` (US-PM-003)

**Auth:** `USER`, `PARTICIPANT`, or `CATALOG`

Paginated (`page`, `size`). Each entry includes the **global participant identifier**, legal name,
self-description URI and contact email. There are no secrets to exclude any more, so the field
selection that existed to hide them is gone.

#### 4. Get Participant by Identifier — `GET /participants/{identifier}` (US-PM-004)

**Auth:** `USER`, `PARTICIPANT`, or `CATALOG`

`{identifier}` is the URL-encoded global identifier. The pre-revision lookup-by-`clientID` path is
removed along with client IDs. 404 if unknown.

#### 5. View My Participant Profile — `GET /participants/me` (US-PM-005)

**Auth:** `PARTICIPANT`

Returns the caller's own full record, resolved from the token.

#### 6. Update My Participant — `PUT /participants/me` (US-PM-006)

**Auth:** `PARTICIPANT`

Updates legal name, self-description URI, contact email, legal person details and the notification
endpoint. The identifier is immutable — a change of identifier is a different organization.

The response is the **re-read** entity, not the in-memory object that was written (original bug
05-PM-12.5 returned stale data).

#### 7. Deregister My Participant — `DELETE /participants/me` (US-PM-007)

**Auth:** `PARTICIPANT`

Cascade semantics are explicit, which the original had not defined at all (original bug 05-PM-12.6):

1. Reject with **409 Conflict** if any consent naming the participant as provider or consumer is
   still `GRANTED`. Revoke or terminate them first — deregistering must not silently strip the basis
   for an active data-sharing arrangement.
2. All `user_participants` links for the participant are removed.
3. All active privacy notices where the participant is provider or consumer are archived.
4. Consents and consent events are **retained**. They are the audit trail; the participant reference
   remains resolvable because the row is retained, not the participant.
5. The participant row is then deleted.

Given `ON DELETE RESTRICT` on the consent foreign keys (TICKET-002), step 1 is enforced by the
database as well as the service.

#### 8. Consent Receipt Verification Keys — `GET /.well-known/jwks.json` (US-PM-008)

**Auth:** none (the keys are public)

Returns the Consent Manager's **JWKS** so participants can verify the signature on consent receipts.
This replaces the pre-revision `GET /participants/consent-signature` endpoint that returned a single
PEM string: a JWKS carries a `kid`, supports rotation, and is consumable by any standard JOSE
library without bespoke parsing.

The key material and signing behaviour are owned by TICKET-011.

### Validation

Bean Validation on the request DTOs:

- `legalName` — `@NotBlank`, max length
- `selfDescriptionUri`, `endpoints.consentNotification` — `@Pattern` for an absolute `https` URI
- `email` — `@Email`

Violations produce an RFC 7807 400 response naming the offending fields (TICKET-015). This replaces
the hand-written Joi middleware and the separate create/update schema files.

### Files to Create

```
src/main/java/eu/prometheusx/consentmanager/
  api/ParticipantController.java
  api/dto/ParticipantResponse.java
  api/dto/ParticipantRegistrationRequest.java
  api/dto/ParticipantUpdateRequest.java
  service/ParticipantService.java
```

## Acceptance Criteria

1. [ ] `POST /participants` creates a record with the identifier taken from the token
2. [ ] An `identifier` supplied in the request body is ignored, not honoured
3. [ ] Registering twice for the same identifier returns 409
4. [ ] `GET /participants` returns a paginated list including the global identifier
5. [ ] `GET /participants/{identifier}` resolves by global identifier and 404s on unknown
6. [ ] `GET /participants/me` returns the caller's record resolved from the token
7. [ ] `PUT /participants/me` updates metadata and returns the re-read entity
8. [ ] `PUT /participants/me` cannot change the identifier
9. [ ] A participant cannot read or modify another participant's record (403)
10. [ ] `DELETE /participants/me` returns 409 while any `GRANTED` consent names the participant
11. [ ] `DELETE /participants/me` removes links, archives privacy notices, and retains consents
12. [ ] `GET /.well-known/jwks.json` returns a valid JWKS with a `kid` and requires no authentication
13. [ ] Bean Validation rejects a blank legal name and a non-`https` notification endpoint with 400
14. [ ] No response anywhere in the module contains a credential field
15. [ ] Parameterized tests cover the validation matrix for registration and update payloads

## Known Issues Resolved by Design

| Original bug | Why it cannot recur |
|--------------|---------------------|
| 05-PM-12.1 — `getMyParticipant` exposes secrets | No secrets are stored |
| 05-PM-12.2 — update `save()` not awaited | Transactional service methods; the write completes or the transaction fails |
| 05-PM-12.3 — `clientSecret` stored in plaintext | No client secrets |
| 05-PM-12.4 — registration `jsonld` dead code | Participant JSON-LD is a stored opaque URI only (TICKET-015) |
| 05-PM-12.5 — DSC integration errors not handled | DSC integration removed entirely |
| 05-PM-12.6 — no cascade on deletion | Explicit cascade policy above, backed by FK constraints |
| 05-PM-12.7 — `validatePassword` declared but absent | No passwords |

## Removed Scope

**US-PM-009 (DSC integration on registration) is deleted.** The Consent Manager no longer fetches a
participant's self-description, logs in to their Data Space Connector, or pushes its public key and
URI to them. Participants retrieve the verification keys themselves from
`GET /.well-known/jwks.json` when they need them.
