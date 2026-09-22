# TICKET-012: Data Exchange

## Metadata

| Field | Value |
|-------|-------|
| **Ticket ID** | TICKET-012 |
| **Title** | Data Exchange |
| **Priority** | High |
| **Complexity** | XL (Extra Large) |
| **Dependencies** | TICKET-005, TICKET-008, TICKET-011 |
| **Related User Stories** | US-DE-001, US-DE-002, US-DE-003, US-DE-004, US-DE-005, US-DE-006, US-CM-017, US-CM-018, US-CM-023 |

## Description

Implement the three-party data exchange protocol between the Consent Manager, Provider DSC (Data Space Connector), and Consumer DSC. This covers user-triggered exchanges, provider-triggered exchanges, inline exchanges triggered during consent grant, the encrypted consent transmission protocol, and token management for verifying exchange authorization.

## Technical Details

### Architecture Overview

```
┌──────────┐     1. Encrypt consent      ┌──────────────┐
│  Consent  │ ──────────────────────────► │ Provider DSC │
│  Manager  │     2. POST consentExport   │              │
│           │ ◄────────────────────────── │              │
│           │     3. Return exchangeId    │              │
│           │                             └──────┬───────┘
│           │                                    │
│           │     4. POST token callback         │
│           │ ◄──────────────────────────────────┘
│           │
│           │     5. Forward to consumer  ┌──────────────┐
│           │ ──────────────────────────► │ Consumer DSC │
│           │     6. POST consentImport   │              │
│           │ ◄────────────────────────── │              │
│           │     7. Token verified        └──────────────┘
└──────────┘
```

### Endpoints

#### 1. Trigger Data Exchange (Inline) — `triggerDataExchangeByConsentId`

**Internal function** (called from consent grant and re-confirm flows)

**Input:** `consentId: string`

**Flow:**
1. Load consent by ID, verify status is `"granted"`
2. Populate provider and consumer participant data
3. Get the provider's `endpoints.consentExport` URL
4. Encrypt the consent using `encryptPayloadAndKey()` with the provider's public key
5. POST encrypted consent to provider's `consentExport` endpoint:
   ```json
   {
     "consentId": "string",
     "encryptedConsent": "base64",
     "encryptedKey": "base64",
     "iv": "base64"
   }
   ```
6. Provider returns `{ dataExchangeId: "string" }`
7. Store the `dataExchangeId` on the consent (or in a separate tracking object)
8. Wait for the provider's token callback (step 4 in diagram)

#### 2. User-Triggered Data Exchange — `POST /consents/:consentId/data-exchange`

**Auth:** `verifyUserJWT`

**Flow:**
1. Find consent by ID
2. Verify the authenticated user owns this consent
3. Verify consent status is `"granted"`
4. Call `triggerDataExchangeByConsentId(consentId)`
5. Return exchange status

**Response:** `200 OK` with `{ message: "Data exchange triggered", dataExchangeId: "string" }`

#### 3. Provider-Triggered Data Exchange — `POST /data-exchange/push`

**Auth:** `verifyParticipantJWT`

**Request Body:**
```json
{
  "consentId": "string (required)",
  "providerDataExchangeId": "string (optional)"
}
```

**Flow:**
1. Verify the authenticated participant is the provider for this consent
2. Find consent by ID
3. Verify consent status is `"granted"`
4. Call `triggerDataExchangeByConsentId(consentId)`
5. Return exchange status

**Response:** `200 OK`

#### 4. User Push Data Exchange — `POST /data-exchange/push/user`

**Auth:** `verifyUserJWT`

**Request Body:**
```json
{
  "consentId": "string (required)"
}
```

**Flow:**
Same as user-triggered exchange above but via the `/data-exchange` route prefix.

#### 5. Attach Token to Consent — `POST /consents/:consentId/token`

**Auth:** `verifyParticipantJWT`

**Request Body:**
```json
{
  "token": "string (required — data exchange authorization token)"
}
```

**Flow:**
1. Find consent by ID
2. Verify the authenticated participant is the provider
3. Store the token on the consent: `consent.token = token`
4. Save consent
5. Forward the token to the consumer's `consentImport` endpoint:
   ```json
   {
     "consentId": "string",
     "token": "string"
   }
   ```
6. Return the consumer's response

**Response:** `200 OK` with forwarded response

This is the "token callback" step — the provider calls this after receiving the encrypted consent, to provide the authorization token that the consumer needs to fetch the actual data.

#### 6. Validate Token — `POST /consents/:consentId/validate`

**Auth:** `verifyParticipantJWT`

**Request Body:**
```json
{
  "token": "string (required)"
}
```

**Flow:**
1. Find consent by ID
2. Verify the authenticated participant is the consumer
3. Compare the provided token against `consent.token`
4. Return validation result

**Response:** `200 OK` with `{ valid: boolean }`

#### 7. Pull Data Exchange — `POST /data-exchange/pull`

**Auth:** (Add authentication — fixing original missing-auth bug)

**Status:** Stub endpoint — not functionally implemented

**Response:** `200 OK` with `{ message: "Not implemented, reserved for future use" }`

### Token Management Protocol

The full 6-step token protocol:

1. **Consent Manager** encrypts consent and POSTs to **Provider DSC** `consentExport`
2. **Provider DSC** returns `dataExchangeId` to Consent Manager
3. **Provider DSC** calls back `POST /consents/:consentId/token` with the authorization token
4. **Consent Manager** stores the token on the consent
5. **Consent Manager** forwards the token to **Consumer DSC** `consentImport`
6. **Consumer DSC** can later call `POST /consents/:consentId/validate` to verify the token

### Error Handling

For each external service call (provider DSC, consumer DSC):
- Catch HTTP errors and return descriptive error responses
- Do not suppress consumer error responses (fixing original bug 06-DE-12.4)
- Log all external service failures
- Return the actual error status to the caller

### Files to Create/Modify

```
src/controllers/dataExchangeController.ts  # Exchange orchestration
src/controllers/consentsController.ts      # Token endpoints
src/routes/dataExchange.ts                 # Data exchange routes
src/routes/consent.ts                      # Token routes
```

## Acceptance Criteria

1. [ ] `triggerDataExchangeByConsentId` encrypts consent and POSTs to provider's `consentExport`
2. [ ] `POST /consents/:consentId/data-exchange` triggers exchange for the authenticated user
3. [ ] `POST /data-exchange/push` triggers exchange for the authenticated provider
4. [ ] `POST /consents/:consentId/token` stores the token and forwards to consumer
5. [ ] `POST /consents/:consentId/validate` verifies the token
6. [ ] `POST /data-exchange/pull` returns a proper stub response with authentication
7. [ ] Inline exchange works when `triggerDataExchange=true` on consent grant
8. [ ] All external service failures are logged and returned (not suppressed)
9. [ ] Token callback correctly forwards to consumer's `consentImport` endpoint
10. [ ] Consumer error responses are not suppressed (actual status is returned)
11. [ ] Integration tests mock DSC endpoints and test the full exchange flow

## Known Issues to Address

1. **`sendConsent` is a stub** (original bug 06-DE-12.1): The standalone `sendConsent` function does nothing useful. Implement proper consent transmission via `triggerDataExchangeByConsentId` for all paths.
2. **Standalone endpoint participant query bug** (original bug 06-DE-12.2): The original queries `Participant.findOne({ assignerID })` which uses a non-existent field. Use proper participant lookup by ID or self-description URL.
3. **Encryption error swallowed** (original bug 06-DE-12.3): Re-throw encryption errors (handled in TICKET-011).
4. **Consumer error suppressed** (original bug 06-DE-12.4): When the POST to consumer's `consentImport` fails, return the actual error to the caller — do not mask it with a 200 success.
5. **Pull exchange has no authentication** (original bug 06-DE-12.5): Add appropriate authentication middleware.
