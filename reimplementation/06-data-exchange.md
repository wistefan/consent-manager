# Data Exchange

This document provides comprehensive feature documentation for the data exchange module. It covers the three-party exchange architecture, user-triggered and provider-triggered flows, inline exchange during consent granting, the encryption protocol, the token-based callback flow, and the pull exchange stub.

> **Source files:** `src/controllers/dataExchangeController.ts`, `src/routes/dataExchange.ts`, `src/controllers/consentsController.ts` (functions: `triggerDataExchangeByConsentId`, `encryptPayloadAndKey`, `attachTokenToConsent`, `verifyToken`), `src/utils/contracts.ts` (functions: `sendConsent`, `validate`, `getParticipantIdentifier`)

---

## Table of Contents

1. [Data Exchange Architecture](#1-data-exchange-architecture)
2. [Standalone Exchange: User-Triggered](#2-standalone-exchange-user-triggered)
3. [Standalone Exchange: Provider-Triggered](#3-standalone-exchange-provider-triggered)
4. [Inline Exchange (Consent Grant)](#4-inline-exchange-consent-grant)
5. [Encryption Protocol](#5-encryption-protocol)
6. [Token Callback Flow](#6-token-callback-flow)
7. [Token Verification](#7-token-verification)
8. [Pull Exchange (Stub)](#8-pull-exchange-stub)
9. [Complete Data Exchange Protocol](#9-complete-data-exchange-protocol)
10. [Standalone vs. Inline Exchange Comparison](#10-standalone-vs-inline-exchange-comparison)
11. [Endpoint Reference](#11-endpoint-reference)
12. [Known Bugs](#12-known-bugs)

---

## 1. Data Exchange Architecture

Data exchange in the Consent Manager follows a **three-party model** involving:

1. **Consent Manager** -- the trust anchor that stores consent, encrypts payloads, and orchestrates the flow
2. **Provider DSC (Data Space Connector)** -- the data provider's connector that exports data and generates exchange tokens
3. **Consumer DSC** -- the data consumer's connector that imports data and verifies tokens

```
User / Participant
        |
        | Trigger (POST /data-exchange/push/user or /push)
        v
+------------------+     1. Encrypted consent      +------------------+
|                  | --------------------------->   |                  |
| Consent Manager  |     POST consentExport         | Provider DSC     |
|                  |                                |                  |
|                  |  <---------------------------  |                  |
|                  |     2. token +                  |                  |
|                  |        providerDataExchangeId   |                  |
|                  |     POST /consents/:id/token    |                  |
|                  |                                +------------------+
|                  |     3. Encrypted consent              |
|                  |        + token + dataExport URL       |
|                  | --------------------------->   +------+----------+
|                  |     POST consentImport         |                  |
+------------------+                                | Consumer DSC     |
        ^                                          |                  |
        |          4. Verify token                  |                  |
        +------ POST /consents/:id/validate <------|                  |
                                                   +------------------+
                                                          |
                                                   5. Pull data from
                                                      Provider's dataExport
```

---

## 2. Standalone Exchange: User-Triggered

**Endpoint:** `POST /data-exchange/push/user` `[UserJWT]`

**Request body:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `consentId` | string | yes | Consent ID (MongoDB ObjectId or string identifier) |

**Logic flow:**

1. Read `req.user.id` from the `verifyUserJWT` middleware.
2. Resolve `consentId` -- if it is a valid ObjectId, query by `_id`; otherwise query by `identifier` field.
3. Find the consent scoped to the authenticated user: `Consent.findOne({ ...query, user: userId })`.
4. Validate the consent: call `consent.isValid()` -- checks that `status === "granted"` and `consented === true`.
5. Extract participant IDs: `consent.getParticipantsIDs()` returns `{ assignerID, assigneeID }`.
6. Fetch both participants in parallel:
   - Provider: `Participant.findOne({ assignerID })`
   - Consumer: `Participant.findOne({ _id: assigneeID })`
7. Build participant identifiers using `getParticipantIdentifier(participant)` -- extracts `consentManager.solution` + `consentManager.idInSolution` from the participant's `jsonld` field and URI-encodes them.
8. Validate the contract between both parties: `validate(providerIdentifier, consumerIdentifier)` -- calls `GET {CONTRACT_SERVICE_BASE_URL}/verify/{providerID}/{consumerID}`.
9. If contract validation fails (status ≠ 200), return `403`.
10. Send consent to the data connector: `sendConsent(consent.jsonData)`.
11. Return success response.

**Response:** `200`
```json
{
  "success": true,
  "message": "consent sent to data connector"
}
```

**Errors:**
- `403` -- `{ error: "Consent has not been granted" }` (consent not valid)
- `403` -- `{ success: false, error: "Invalid contract" }` (contract verification failed)
- `NotFoundError` -- Consent not found (→ 404 via error handler)

> **⚠ Bug:** The `sendConsent` function is a stub that does not actually send data. See [Known Bugs](#121-sendconsent-is-a-stub).

---

## 3. Standalone Exchange: Provider-Triggered

**Endpoint:** `POST /data-exchange/push` `[ParticipantJWT]`

**Request body:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `consentId` | string | yes | Consent ID (MongoDB ObjectId or string identifier) |
| `userId` | string | yes | UserIdentifier ID scoped to the authenticated participant |

**Logic flow:**

1. Read `req.userParticipant.id` from the `verifyParticipantJWT` middleware.
2. Find the UserIdentifier scoped to the authenticated participant: `UserIdentifier.findOne({ attachedParticipant: req.userParticipant.id, _id: userId })`.
3. If not found, throw `NotFoundError("User Identifier not found")`.
4. Resolve `consentId` (ObjectId or identifier string) -- same as user-triggered step 2.
5. Find the consent scoped to the resolved user: `Consent.findOne({ ...query, user: userIdentifier.user })` -- same as user-triggered step 3, but uses `userIdentifier.user` instead of the authenticated `userId`.
6. From here, steps 4-11 of the user-triggered flow above apply unchanged (validate consent, extract participant IDs, fetch participants, build identifiers, validate contract, send consent, return success).

**Response:** `200`
```json
{
  "success": true,
  "message": "consent sent to data connector"
}
```

**Key difference from user-triggered:** The provider identifies the user via its own `UserIdentifier` mapping (`userId` in the request body), rather than the user authenticating directly. This allows providers to initiate exchanges on behalf of their users.

---

## 4. Inline Exchange (Consent Grant)

When a consent is granted with the query parameter `?triggerDataExchange=true`, the system triggers a data exchange immediately as part of the consent grant response.

### 4.1 Trigger Points

The internal function `triggerDataExchangeByConsentId` is called from several places in `src/controllers/consentsController.ts`:

| Caller | Context |
|--------|---------|
| `giveConsent` | User Key consent grant with `triggerDataExchange=true` |
| `giveConsentUser` | User JWT consent grant with `triggerDataExchange=true` |
| `giveConsentOnMultipleDataExchange` | Multi-exchange consent grant |
| `triggerDataExchange` (exported controller) | Explicit `POST /consents/:consentId/data-exchange` call |
| `reConfirmConsent` | Consent re-confirmation with `triggerDataExchange=true` |
| Email verification callback | When email-verified consent has `triggerDataExchange=true` |

### 4.2 `triggerDataExchangeByConsentId` Function

**Signature:** `triggerDataExchangeByConsentId(consentId: string, res: Response)` *(private, non-exported)*

**Logic flow:**

1. Fetch the consent by ID, populating `dataConsumer` and `dataProvider` (excluding `clientID` and `clientSecret` from populated documents).
2. If consent not found, return `404`.
3. If `consent.status !== "granted"`, return `401`.
4. Clone the consent object as the payload.
5. Encrypt the payload using `encryptPayloadAndKey(payload)` (see [Section 5](#5-encryption-protocol)):
   - AES-256-CBC encrypts the consent JSON.
   - RSA encrypts the AES key.
6. POST `{ signedConsent, encrypted }` to `consent.dataProvider.endpoints.consentExport`.
7. Convert the consent to a consent receipt using `consentToConsentReceipt()`.
8. Return the consent receipt and the `dataExchangeId` from the provider's response.

**Response:** `200`
```json
{
  "message": "Successfully sent consent to the provider's consent export endpoint to trigger the data exchange",
  "consentReceipt": { /* ISO/IEC 29184 receipt */ },
  "dataExchangeId": "<from provider response>"
}
```

**Error handling:**
- Axios response error (provider unreachable): `424` -- `{ error: "Failed to communicate with the data consumer connector" }`
- Other errors: `500` -- `{ error: "Internal server error" }`

---

## 5. Encryption Protocol

The Consent Manager uses a hybrid encryption scheme combining AES-256-CBC (for bulk encryption) and RSA (for key encryption) to secure consent payloads sent to participant DSCs.

### 5.1 Key Files

| File | Type | Loaded At | Purpose |
|------|------|-----------|---------|
| `src/config/keys/consentSignature.pem` | RSA private key | Module import (synchronous) | Encrypting the AES key |
| `src/config/keys/AESKey.pem` | AES-256 key | Module import (synchronous) | Encrypting consent payloads |
| `src/config/keys/consentSignaturePublic.pem` | RSA public key | On-demand | Shared with participants for verification |

### 5.2 `encryptPayloadAndKey` Function

**Location:** `src/controllers/consentsController.ts`

**Input:** `payload: object` (the consent document to encrypt)

**Step 1 -- AES Encryption of the Consent Payload:**

1. Generate a random 16-byte initialization vector (IV) using `crypto.randomBytes(16)`.
2. Create an AES-256-CBC cipher with the AES key and IV.
3. Encrypt `JSON.stringify(payload)` using the cipher.
4. Produce `signedConsent` as `"<iv_hex>:<ciphertext_hex>"` (IV and ciphertext separated by colon, both hex-encoded).

**Step 2 -- RSA Encryption of the AES Key:**

1. Load the RSA private key from PEM with an empty passphrase.
2. Encrypt the AES key buffer using `crypto.privateEncrypt` with `RSA_PKCS1_PADDING`.
3. Produce `encrypted` as a Buffer (the RSA-encrypted AES key).

**Output:**
```typescript
{
  signedConsent: "<iv_hex>:<ciphertext_hex>",  // AES-encrypted consent
  encrypted: Buffer  // RSA-encrypted AES key
}
```

### 5.3 Decryption (Receiver Side)

The receiving DSC must:

1. Decrypt the AES key using the Consent Manager's RSA **public key** (obtained during registration or via `GET /participants/consent-signature`).
2. Split `signedConsent` on `:` to extract the IV (first part) and ciphertext (second part).
3. Decode both from hex.
4. Create an AES-256-CBC decipher with the decrypted AES key and the IV.
5. Decrypt the ciphertext to obtain the JSON consent payload.

> **Note:** The function uses `crypto.privateEncrypt` (not `sign`), which means the "encryption" is technically RSA signing of the AES key. The receiver uses `publicDecrypt` to recover it. This is a non-standard but functional approach to authenticated key exchange.

---

## 6. Token Callback Flow

After the Consent Manager sends the encrypted consent to the provider's `consentExport` endpoint, the provider generates a token and calls back.

### 6.1 Attach Token to Consent

**Endpoint:** `POST /consents/:consentId/token` `[ParticipantJWT]`

**Request body:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `token` | string | yes | Provider-generated authentication token |
| `providerDataExchangeId` | string | yes | Provider's data exchange identifier |

**Logic flow:**

1. Find the consent by `consentId`, populating `dataConsumer`, `dataProvider`, `providerUserIdentifier`, and `consumerUserIdentifier`.
2. If not found, return `404`.
3. Update the consent document: `Consent.findByIdAndUpdate(consentId, { token }, { new: true })` with full population.
4. Build a combined payload: the updated consent plus `providerDataExchangeId`.
5. Encrypt the payload using `encryptPayloadAndKey(payload)`.
6. Forward the encrypted consent to the consumer's import endpoint:
   ```
   POST <consent.dataConsumer.endpoints.consentImport>
   {
     "dataProviderEndpoint": "<consent.dataProvider.endpoints.dataExport>",
     "signedConsent": "<encrypted consent>",
     "encrypted": "<encrypted AES key>"
   }
   ```
7. Return success response.

**Response:** `200`
```json
{ "message": "successfully forwarded consent to the data consumer" }
```

**Errors:**
- `404` -- `{ error: "Consent not found" }`
- Errors from the consumer's import endpoint are caught and logged but do not change the response.

---

## 7. Token Verification

**Endpoint:** `POST /consents/:consentId/validate` `[ParticipantJWT]`

**Request body:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `token` | string | yes | Token to verify against the stored consent token |

**Logic flow:**

1. Find the consent by `consentId`.
2. Compare the provided `token` against `consent.token`.
3. Return verification result.

**Response (success):** `200`
```json
{ "message": "token matches consent token", "verified": true }
```

**Response (failure):** `400`
```json
{ "error": "token does not match consent token" }
```

**Errors:**
- `404` -- `{ error: "Consent not found" }`

---

## 8. Pull Exchange (Stub)

**Endpoint:** `POST /data-exchange/pull` `[None]`

**Status:** Not implemented.

**Response:** `200`
```json
{ "message": "Not implemented, kept for improvements" }
```

> **Note:** This endpoint has no authentication middleware and no request validation. It exists as a placeholder for future pull-based data exchange functionality.

---

## 9. Complete Data Exchange Protocol

The full data exchange protocol for the inline (functional) path works as follows:

### Step 1: Trigger

A data exchange is triggered by one of:
- User granting consent with `?triggerDataExchange=true`
- User calling `POST /consents/:consentId/data-exchange`
- User calling `POST /data-exchange/push/user` (via standalone, but currently uses stub)
- Provider calling `POST /data-exchange/push` (via standalone, but currently uses stub)
- User re-confirming consent with `?triggerDataExchange=true`

### Step 2: Consent Manager → Provider DSC

The Consent Manager:
1. Loads the consent with populated participant documents.
2. Verifies the consent is granted.
3. Encrypts the consent (AES-256-CBC for payload, RSA for the AES key).
4. Sends `{ signedConsent, encrypted }` to the provider's `consentExport` endpoint.

### Step 3: Provider DSC → Consent Manager

The Provider DSC:
1. Decrypts the consent using the Consent Manager's public key.
2. Generates a unique token and a `providerDataExchangeId`.
3. Calls `POST /consents/:consentId/token` on the Consent Manager with `{ token, providerDataExchangeId }`.

### Step 4: Consent Manager → Consumer DSC

The Consent Manager:
1. Stores the token on the consent document.
2. Encrypts the updated consent (with token and `providerDataExchangeId`).
3. Forwards to the consumer's `consentImport` endpoint with:
   - The encrypted consent
   - The provider's `dataExport` URL

### Step 5: Consumer DSC → Consent Manager (Optional)

The Consumer DSC:
1. Can call `POST /consents/:consentId/validate` with the token to verify its authenticity.

### Step 6: Consumer DSC → Provider DSC

The Consumer DSC:
1. Uses the `dataProviderEndpoint` (dataExport URL) from step 4 to pull data directly from the provider.
2. The token serves as the authentication/authorization credential for the data transfer.

---

## 10. Standalone vs. Inline Exchange Comparison

| Aspect | Standalone (`/data-exchange/push/*`) | Inline (`triggerDataExchangeByConsentId`) |
|--------|--------------------------------------|------------------------------------------|
| **Trigger** | Explicit API call | During consent grant/reconfirm |
| **Encryption** | None (uses `sendConsent` stub) | AES-256-CBC + RSA |
| **Contract Validation** | Yes (via `validate()`) | No (consent already validated) |
| **Provider Communication** | Stub (`sendConsent` does nothing) | Real (POSTs to `consentExport`) |
| **Response** | Simple success message | Consent receipt + `dataExchangeId` |
| **Functional** | **No** (stub) | **Yes** (real exchange) |
| **Token Flow** | Not initiated | Full three-party token flow |

> **⚠ Important:** The standalone exchange endpoints (`/data-exchange/push/*`) are **non-functional** in the current implementation. The `sendConsent` function in `src/utils/contracts.ts` performs a `GET /` and overwrites the response data locally, which has no external effect. The inline exchange path is the only working data exchange mechanism.

---

## 11. Endpoint Reference

| # | Method | Path | Auth | Controller | Description |
|---|--------|------|------|------------|-------------|
| 1 | `POST` | `/data-exchange/push/user` | UserJWT | `userTriggeredDataExchange` | User triggers data exchange (standalone, **stub**) |
| 2 | `POST` | `/data-exchange/push` | ParticipantJWT | `providerTriggeredDataExchange` | Provider triggers data exchange (standalone, **stub**) |
| 3 | `POST` | `/data-exchange/pull` | None | *(inline)* | Pull exchange (**not implemented**) |
| 4 | `POST` | `/consents/:consentId/data-exchange` | UserJWT | `triggerDataExchange` | User triggers inline data exchange |
| 5 | `POST` | `/consents/:consentId/token` | ParticipantJWT | `attachTokenToConsent` | Provider attaches token (callback) |
| 6 | `POST` | `/consents/:consentId/validate` | ParticipantJWT | `verifyToken` | Validate token against consent |

> **Note:** Endpoints 4-6 are defined in `src/routes/consent.ts` and are part of the consent routes, not the data exchange routes. They are listed here because they are integral to the data exchange protocol.

---

## 12. Known Bugs

### 12.1 `sendConsent` Is a Stub

**Location:** `src/utils/contracts.ts` (`sendConsent` function)

**Description:** The `sendConsent` function is supposed to send consent data to the data connector, but the implementation is:
```typescript
export const sendConsent = async (consentSD: any) => {
  const res = await axios.get("/");
  res.data = { success: true, consent: consentSD };
};
```
It performs a `GET /` (to localhost) and overwrites `res.data` locally, which has no external effect. The consent data is never actually sent anywhere.

**Impact:** The standalone data exchange endpoints (`POST /data-exchange/push/user` and `POST /data-exchange/push`) appear to succeed but perform no actual data exchange. Only the inline path (via `triggerDataExchangeByConsentId`) actually communicates with participant DSCs.

### 12.2 Standalone Endpoints Participant Query Bug

**Location:** `src/controllers/dataExchangeController.ts` (both functions)

**Description:** The provider participant is queried as `Participant.findOne({ assignerID })` where `assignerID` is a URL string. However, `assignerID` is not a field on the Participant schema -- it is a value from the consent's permission structure. This query will always return `null` because it searches for a field that does not exist. The consumer lookup `Participant.findOne({ _id: assigneeID })` is correct.

**Impact:** Even if `sendConsent` were functional, the standalone exchange would fail at the contract validation step because `dataProvider` would be `null`, causing `getParticipantIdentifier` to throw `NotFoundError("Participant not found")`.

### 12.3 `encryptPayloadAndKey` Error Swallowed

**Location:** `src/controllers/consentsController.ts` (`encryptPayloadAndKey` function)

**Description:** The try/catch block in `encryptPayloadAndKey` catches all errors and only logs them via `Logger.error(e)`. It does not rethrow the error or return an error indicator. On encryption failure, the function returns `undefined`, which would cause downstream code to fail with a confusing error when trying to access `signedConsent` and `encrypted` properties of `undefined`.

### 12.4 Consumer Error Response Suppressed in Token Callback

**Location:** `src/controllers/consentsController.ts` (`attachTokenToConsent` function)

**Description:** When the POST to the consumer's `consentImport` endpoint fails, the error is caught, logged, and **suppressed**. The function still returns `200` with "successfully forwarded consent to the data consumer" even though the forwarding failed. The caller (the provider DSC) receives no indication that the consumer did not receive the consent.

### 12.5 Pull Exchange Has No Authentication

**Location:** `src/routes/dataExchange.ts` (pull route)

**Description:** The `POST /data-exchange/pull` endpoint has no authentication middleware. While the endpoint is a stub returning a static message, the lack of auth is a concern if the endpoint is implemented in the future without adding middleware.
