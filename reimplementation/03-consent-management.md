# Consent Management

This document provides comprehensive feature documentation for the consent management module -- the largest and most complex part of the Consent Manager. It covers the consent lifecycle state machine, privacy notice generation, all consent grant flows, lifecycle operations, querying, receipt formats, encryption, and token management.

> **Source files:** `src/controllers/consentsController.ts` (~2200 lines), `src/routes/consent.ts`, `src/models/Consent/Consent.model.ts`, `src/models/PrivacyNotice/PrivacyNotice.model.ts`, `src/utils/consentReceipt.ts`, `src/utils/consentEvent.ts`, `src/libs/consent-generator/`

---

## Table of Contents

1. [Consent Lifecycle State Machine](#1-consent-lifecycle-state-machine)
2. [Privacy Notice Generation](#2-privacy-notice-generation)
3. [Consent Grant Flow (User Key)](#3-consent-grant-flow-user-key)
4. [Consent Grant Flow (User JWT)](#4-consent-grant-flow-user-jwt)
5. [Email Verification Flow](#5-email-verification-flow)
6. [Consent Revocation](#6-consent-revocation)
7. [Consent Refusal](#7-consent-refusal)
8. [Consent Termination](#8-consent-termination)
9. [Consent Re-confirmation](#9-consent-re-confirmation)
10. [Consent Resume](#10-consent-resume)
11. [Consent Receipt Formats](#11-consent-receipt-formats)
12. [Consent Querying](#12-consent-querying)
13. [PDI Iframe Redirect](#13-pdi-iframe-redirect)
14. [Available Exchanges](#14-available-exchanges)
15. [Token Management](#15-token-management)
16. [Consent Encryption](#16-consent-encryption)
17. [Endpoint Reference](#17-endpoint-reference)
18. [Known Bugs](#18-known-bugs)

---

## 1. Consent Lifecycle State Machine

Consents follow a defined state machine with seven possible states. State transitions are triggered by specific API operations.

### 1.1 States

| State | Meaning |
|-------|---------|
| `pending` | Consent created but waiting for the consumer-side user to be registered (consumer app registration flow) |
| `draft` | Same as pending but marked for automatic data exchange trigger once the consumer-side user is established |
| `granted` | User has actively consented; data exchange is authorized |
| `revoked` | User has withdrawn a previously granted consent |
| `expired` | Consent has exceeded its validity period (not triggered by any current endpoint -- reserved for future TTL-based expiration) |
| `terminated` | Consent has been terminated (typically by the user or system, distinct from revocation) |
| `refused` | User explicitly declined to give consent |

### 1.2 State Transition Diagram

```
                  registerNewUserToConsumerSide
                  (triggerDataExchange=false)
                           |
                           v
                      +---------+
            +-------->| pending |--------+
            |         +---------+        |
            |              |             |
            |  resumeConsent             | refuseConsent / terminateConsent
            |              |             |
            |              v             v
            |         +---------+   +---------+   +---------+
   (new) ---+-------->|  draft  |   | refused |   |terminated|
            |         +---------+   +---------+   +---------+
            |              |
            |  resumeConsent
            |              |
            |              v
            |         +---------+
            +-------->| granted |
                      +---------+
                       /    |    \
          revokeConsent  refuseConsent  terminateConsent
                     /      |      \
                    v       v       v
            +---------+ +---------+ +-----------+
            | revoked | | refused | | terminated|
            +---------+ +---------+ +-----------+
```

### 1.3 Transitions

| From | To | Trigger | Endpoint |
|------|----|---------|----------|
| *(new)* | `pending` | Consumer user not yet registered, no data exchange trigger | `POST /consents` (internal `registerNewUserToConsumerSide`) |
| *(new)* | `draft` | Consumer user not yet registered, data exchange trigger requested | `POST /consents` (internal `registerNewUserToConsumerSide`) |
| *(new)* | `granted` | User gives consent (both identifiers resolved) | `POST /consents`, `POST /consents/user`, `GET /consents/emailverification` |
| *(new)* | `refused` | User refuses consent at creation time | `POST /consents` with `event="refused"` |
| `pending` | `granted` | Consumer provides user identity | `POST /consents/:consentId/resume` |
| `draft` | `granted` | Consumer provides user identity (auto-triggers data exchange) | `POST /consents/:consentId/resume` |
| `granted` | `revoked` | User withdraws consent | `DELETE /consents/:id` |
| `granted`, `pending`, `draft` | `refused` | User refuses consent | `POST /consents/:consentId/refuse` |
| `granted`, `pending`, `draft` | `terminated` | User or system terminates consent | `POST /consents/:consentId/terminate` |

### 1.4 Consent Events

Each consent maintains an `event` array as an audit trail. Each entry has the following structure:

| Field | Type | Description |
|-------|------|-------------|
| `eventTime` | String (ISO 8601) | Timestamp when the event occurred |
| `validityDuration` | String | Always `"0"` (no time-limited consent in current implementation) |
| `eventType` | String | Always `"explicit"` (all transitions are user- or system-initiated) |
| `eventState` | String | Human-readable description of the event |

**Event types:**

| Key | `eventState` Value |
|-----|--------------------|
| `given` | `"consent given"` |
| `revoked` | `"consent revoked"` |
| `refused` | `"consent refused"` |
| `reConfirmed` | `"consent re-confirmed"` |
| `terminated` | `"consent terminated"` |

### 1.5 Parent/Child Consent Linking

When a user re-grants consent on a privacy notice that has a previously revoked or terminated consent:

1. The system queries for an existing consent on the same privacy notice with status `revoked` or `terminated`.
2. The new consent's `parent` field is set to the old consent's `_id`.
3. The old consent's `child` field is updated to point to the new consent's `_id`.

This forms a singly-linked chain of consent history for a given data exchange agreement.

---

## 2. Privacy Notice Generation

Privacy notices are the bridge between contracts (negotiated in the external Contract Service) and user-facing consent. They describe what data is being processed, by whom, for what purpose, and under what conditions.

### 2.1 Fetching Privacy Notices Between Parties

**Endpoint:** `GET /consents/:userId/:providerId/:consumerId` `[UserKey]`

**Parameters:**
- `userId` -- UserIdentifier ID (validated to exist)
- `providerId` -- Base64-encoded self-description URL of the data provider
- `consumerId` -- Base64-encoded self-description URL of the data consumer

**Logic flow:**

1. Validate that `userId` exists as a UserIdentifier.
2. Call `getPrivacyNoticesFromContractsBetweenParties(providerId, consumerId)` which:
   - Fetches **bilateral contracts** from `{CONTRACT_SERVICE_BASE_URL}/bilaterals?hasMember={providerUrl}&hasMember={consumerUrl}`
   - Fetches **ecosystem contracts** from `{CONTRACT_SERVICE_BASE_URL}/contracts?hasMember={providerUrl}&hasMember={consumerUrl}`
   - Converts each contract to a `PrivacyNotice` structure by extracting purposes, data resources, recipients, and controller details from the contract and its service offerings
3. Load existing non-archived (`archivedAt: null`) PrivacyNotice documents from MongoDB for the same provider and consumer.
4. Compare contract IDs between newly fetched and existing notices:
   - **New contracts** (not yet in DB): Create and save new PrivacyNotice documents (only if they have both `data` and `purposes` arrays).
   - **Changed contracts** (data fields differ): Create a new version of the notice and archive the old one by setting `archivedAt` to the current ISO timestamp.
   - **Unchanged contracts**: Keep existing notices.
5. Return the final set of non-archived privacy notices.

**Response:** `200` with array of privacy notices, or `404` if no contracts are found.

### 2.2 Fetching Privacy Notices by Contract URI

**Endpoint:** `GET /consents/:userId/:providerURI/:consumerURI/:contractURI` `[UserJWT]`

**Parameters:** `providerURI`, `consumerURI`, `contractURI` -- all Base64-encoded URLs.

Same logic as above but calls `getPrivacyNoticesFromContract()` for a specific single contract URI. Used when the user already knows the exact contract.

### 2.3 Retrieving a Single Privacy Notice

**Endpoint:** `GET /consents/privacy-notices/:privacyNoticeId` `[UserKey]`
**Endpoint:** `GET /consents/privacy-notices/:privacyNoticeId/user` `[UserJWT]`

**Logic:**

1. Find the PrivacyNotice by `_id`.
2. Check whether the authenticated user has a consumer-side UserIdentifier with the same email (determines whether email input is needed in the frontend).
3. Call `populatePrivacyNotice(pn)` which fetches via HTTP:
   - The contract document from the Contract Service
   - Data resource self-descriptions for each item in `data[]`
   - Recipient self-descriptions for each item in `recipients[]`
   - Purpose self-descriptions for each item in `purposes[]`
   - Replaces URI strings with the fetched full data objects
4. Return the populated notice with a `consumerEmail` boolean flag (`true` when the user does NOT have a consumer-side identifier -- meaning email entry will be required for consent).

**Response:** `200` with populated privacy notice object plus `consumerEmail` flag.

### 2.4 Privacy Notice Archival

Privacy notices are soft-deleted rather than hard-deleted. When a contract's data resources change:

1. A new PrivacyNotice document is created with the updated data.
2. The old notice's `archivedAt` field is set to the current ISO timestamp.
3. All queries filter by `archivedAt: null` to only return active notices.

This preserves the audit trail -- existing consents that reference an archived notice retain their association.

---

## 3. Consent Grant Flow (User Key)

**Endpoint:** `POST /consents` `[UserKey]`

This is the primary consent creation endpoint, called by a participant on behalf of a user. Authentication is via the `x-user-key` header which identifies the provider-side UserIdentifier.

### 3.1 Request

**Body:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `privacyNoticeId` | String | Yes | ID of the privacy notice being consented to |
| `email` | String | No | User's email (needed for cross-participant reconciliation) |
| `event` | String | No | `"given"` (default) or `"refused"` |
| `data` | Array\<String\> | No | Subset of data resource URIs to consent to (filters the notice's full data set) |
| `dataProcessingId` | String | No | Specific data processing ID to consent to |

**Query parameters:**

| Param | Type | Description |
|-------|------|-------------|
| `triggerDataExchange` | String | If truthy, triggers a data exchange immediately after consent creation |

### 3.2 Detailed Logic Flow

```
POST /consents
  |
  v
1. Load provider UserIdentifier from x-user-key header
  |
  v
2. Load PrivacyNotice by privacyNoticeId
  |
  v
3. Resolve dataProvider and dataConsumer Participants
   from the notice's self-description URLs
  |
  v
4. Look for a consumer UserIdentifier with the same
   email as the provider identifier
  |
  v
5. Fetch the consumer's first purpose service offering
   to check the userInteraction flag
  |
  +-- Consumer identifier found? -----> Go to step 8
  |
  v
6. userInteraction flag is true?
  |   |
  |   Yes --> Call userInteraction() helper:
  |           - Create new consumer UserIdentifier
  |           - Login to consumer DSC
  |           - Register user at DSC /private/users/register
  |           - Go to step 8
  |
  v
7. No consumer identifier AND no email provided?
  |   |
  |   Yes --> Call registerNewUserToConsumerSide():
  |           - Create draft/pending consent
  |           - Login to consumer DSC
  |           - POST to DSC /private/users/app
  |           - Return draft/pending consent (DONE)
  |
  |   No --> Email was provided:
  |           Call emailReattached():
  |           - Implements 4-case email reconciliation
  |           - May send verification email (DONE for async cases)
  |           - Or resolves immediately for case 2.c1
  |
  v
8. Both provider and consumer identifiers exist
   Find or create User linking both identifiers
  |
  v
9. Check for duplicate: existing active consent
   (status not in terminated/revoked/refused) with
   matching privacyNotice + user + provider + consumer
  |
  +-- Duplicate found --> Return existing consent (200)
  |                       (or trigger data exchange if requested)
  |
  v
10. Check for parent: revoked/terminated consent on
    same privacy notice for parent/child linking
  |
  v
11. Create consent:
    - event="refused" --> status: "refused", consented: false
    - event="given"   --> status: "granted", consented: true
    - Set parent field if found in step 10
    - Update parent's child field to point to new consent
  |
  v
12. Return consent receipt (201)
    OR trigger data exchange if triggerDataExchange is set
```

### 3.3 Response

- **201** -- New consent created (returns consent receipt)
- **200** -- Existing duplicate consent found (returns consent receipt)
- **200** -- Data exchange triggered (returns consent receipt + `dataExchangeId`)
- **200** -- Email verification requested (returns `{ case: "email-validation-requested", status: 200 }`)
- **200** -- Draft/pending consent created (returns consent)
- **400** -- Empty `data` array, missing `privacyNoticeId`, or invalid `event` value
- **404** -- Privacy notice, consumer participant, or user not found

---

## 4. Consent Grant Flow (User JWT)

**Endpoint:** `POST /consents/user` `[UserJWT]`

Same purpose and request body as the User Key flow, but for directly authenticated users (via JWT).

### 4.1 Key Differences from User Key Flow

1. **User identity source:** Gets `userId` from JWT token (`req.user.id`). Loads the User document with populated `identifiers` array.
2. **Identifier resolution:** Finds provider/consumer identifiers from the user's own identifiers, with auto-reattachment if found by email match.
3. **Same decision tree:** Applies the same `userInteraction`, `registerNewUserToConsumerSide`, `emailReattached` flow as the User Key endpoint.

> **Warning: Known Bug**
> When `event="refused"`, the consent is created with `status: "granted"` and `consented: true` instead of `status: "refused"` and `consented: false`. This is a copy-paste error. The event array correctly records `consentEvent.refused`, but the status and consented fields are wrong. See [Known Bugs](#18-known-bugs) for details.

> **Warning: Known Bug**
> The parent consent update uses `$push: { children: newConsent._id }` (plural "children") instead of `{ child: newConsent._id }` (singular) as in the User Key flow. The Consent schema only defines a `child` field (singular), so this creates an unschema'd field and does not properly link the parent to the child. See [Known Bugs](#18-known-bugs) for details.

---

## 5. Email Verification Flow

**Callback Endpoint:** `GET /consents/emailverification` `[None]`

This endpoint is the callback target for email verification links sent during the consent grant flow. It has **no authentication middleware** -- it is a public URL clicked from the user's email inbox.

### 5.1 When Email Verification is Triggered

Email verification is triggered during the consent grant flow (in `emailReattached()`) when:
- The consumer-side UserIdentifier cannot be found automatically
- An email address is provided
- The user accounts need cross-participant reconciliation

### 5.2 The Four Reconciliation Cases

The `emailReattached()` function implements a decision tree with four cases for reconciling user identities across participants:

#### Case 2.a: Provider has User, Consumer has no User

- **Condition:** The provider UserIdentifier is linked to a User (`providerUser` exists). The consumer UserIdentifier exists (found by email) but is NOT linked to any User.
- **Action:** Send verification email.
- **On email click:** `giveConsentOnEmailValidation` adds the `consumerUserIdentifier` to the provider user's `identifiers` array. The provider user now spans both participants.

#### Case 2.b: Provider has no User, Consumer has User

- **Condition:** The provider UserIdentifier is NOT linked to any User. The consumer UserIdentifier has a linked User (`consumerUser` exists).
- **Action:** Send verification email.
- **On email click:** `giveConsentOnEmailValidation` adds the `providerUserIdentifier` to the consumer user's `identifiers` array. The consumer user now spans both participants.

#### Case 2.c: Both have Users (two sub-cases)

**Case 2.c1: One user was manually registered (has password)**
- **Condition:** Both identifiers have different Users, and one of them has a `password` field (indicating manual registration through the signup flow).
- **Action:** The manually-registered user absorbs the other user's identifier. **No email is sent** -- this resolves immediately.
- **Return:** `{ case: "user-manual-registration-found", status: 200 }`

**Case 2.c2: Neither user was manually registered**
- **Condition:** Both identifiers have different Users, neither has a password.
- **Action:** The provider user is deleted. A verification email is sent.
- **On email click:** `giveConsentOnEmailValidation` adds the `providerUserIdentifier` to the consumer user's `identifiers` array.

#### Case 2.d: Neither has a User

- **Condition:** Neither the provider nor the consumer UserIdentifier is linked to a User.
- **Action:** Send verification email.
- **On email click:** `giveConsentOnEmailValidation` finds or creates a User by email from the consumer identifier, then links both identifiers to that user.

### 5.3 Email Verification Callback

**Query parameters passed in the verification URL:**

| Parameter | Description |
|-----------|-------------|
| `dataProvider` | Provider participant ID |
| `dataConsumer` | Consumer participant ID |
| `privacyNotice` | Privacy notice ID |
| `data` | Consented data resource URIs |
| `triggerDataExchange` | Whether to trigger exchange after consent |
| `providerUserIdentifier` | Provider-side UserIdentifier ID |
| `consumerUserIdentifier` | Consumer-side UserIdentifier ID |
| `providerUser` | Provider-side User ID (if exists) |
| `consumerUser` | Consumer-side User ID (if exists) |
| `dataProcessingId` | Data processing ID (if provided) |

**Callback logic:**

1. Decode all URI-encoded parameters.
2. Check for duplicate (already granted consent with matching parameters). Return existing if found.
3. Apply the appropriate reconciliation case (2.a, 2.b, 2.c2, or 2.d) to link user identifiers.
4. Create consent with `status: "granted"` and `consented: true`.
5. If `triggerDataExchange` is set, encrypt and send consent to the provider's `consentExport` endpoint.

**Response:** `200` with success message, or `200` with data exchange confirmation.

> **Security Note:** This endpoint has no authentication. All consent parameters (participant IDs, user identifiers, privacy notice ID) are passed as query string parameters in the email link. A reimplementation should consider adding HMAC-signed tokens or one-time-use verification codes to prevent URL manipulation.

---

## 6. Consent Revocation

**Endpoint:** `DELETE /consents/:id` `[UserJWT]`

### 6.1 Logic

1. Find consent by `_id` where the `user` field matches the authenticated user.
2. Set `consented: false`.
3. Set `status: "revoked"`.
4. Push `consentEvent.revoked` to the `event` array.
5. Save the consent.

### 6.2 Response

- **200** -- Consent receipt of the revoked consent
- **404** -- Consent not found or not owned by the authenticated user

---

## 7. Consent Refusal

**Endpoint:** `POST /consents/:consentId/refuse` `[UserJWT]`

### 7.1 Logic

1. Find consent by `_id` where status is NOT already `terminated`, `revoked`, or `refused`.
2. Set `status: "refused"`.
3. Push `consentEvent.refused` to the `event` array.
4. Save the consent.

> **Note:** In the original implementation, `consent.save()` is called without `await`, meaning the response is sent before the save operation completes.

### 7.2 Response

- **200** -- Consent receipt of the refused consent
- **404** -- Consent not found or already in a terminal state

---

## 8. Consent Termination

**Endpoint:** `POST /consents/:consentId/terminate` `[UserJWT]`

### 8.1 Logic

1. Find consent by `_id` where status is NOT already `terminated`, `revoked`, or `refused`.
2. Set `status: "terminated"`.
3. Push `consentEvent.terminated` to the `event` array.
4. Save the consent.

> **Note:** Same `await`-less `consent.save()` issue as `refuseConsent`.

### 8.2 Response

- **200** -- Consent receipt of the terminated consent
- **404** -- Consent not found or already in a terminal state

---

## 9. Consent Re-confirmation

**Endpoint:** `POST /consents/:consentId/re-confirm` `[UserJWT]`

### 9.1 Logic

1. Find consent by `_id` where status is NOT `terminated`, `revoked`, or `refused`.
2. Push `consentEvent.reConfirmed` to the `event` array.
3. Save the consent.
4. If `triggerDataExchange` query parameter is set, trigger a data exchange for this consent.

### 9.2 Request

**Query parameters:**

| Param | Type | Description |
|-------|------|-------------|
| `triggerDataExchange` | String | If truthy, triggers a data exchange after re-confirmation |

### 9.3 Response

- **200** -- Consent receipt (or consent receipt + `dataExchangeId` if data exchange triggered)
- **404** -- Consent not found or in a terminal state

---

## 10. Consent Resume

**Endpoint:** `POST /consents/:consentId/resume` `[ParticipantJWT]`

Resumes a pending or draft consent by supplying the consumer-side user identity that was missing at creation time.

### 10.1 Request

**Body:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `internalID` | String | Yes | Internal identifier for the user at the consumer participant |
| `email` | String | Yes | User's email address |

### 10.2 Logic

1. Load consent and verify status is `"pending"` or `"draft"`.
2. Create a new UserIdentifier for the consumer participant:
   - `attachedParticipant`: the data consumer participant ID from the consent
   - `email`: from request body
   - `identifier`: `internalID` from request body
3. Call `checkUserIdentifier()` to reconcile user accounts across participants.
4. **If `"draft"` status:**
   - Set status to `"granted"`.
   - Trigger a data exchange (calls `dataExchanges()`).
5. **If `"pending"` status:**
   - Set status to `"granted"`.
   - Save the consent.

### 10.3 Response

- **200** -- Consent receipt of the now-granted consent
- **404** -- Consent not found
- **400** -- Consent is not in pending or draft status

---

## 11. Consent Receipt Formats

The system supports two consent receipt formats.

### 11.1 ISO/IEC 29184 Consent Receipt (`consentToConsentReceipt`)

This is the primary receipt format, used by all consent query endpoints. Defined in `src/utils/consentReceipt.ts`.

**Transformation process:**

1. Load the consumer and provider Participant documents.
2. Fetch each participant's self-description via HTTP (`GET {participant.selfDescriptionURL}`).
3. If third-party recipients exist (`consent.recipientThirdParties.catalogId`), fetch each recipient participant's self-description.
4. Build the receipt structure.

**Receipt structure:**

```json
{
  "record": {
    "schemaVersion": "<consent.schema_version>",
    "recordId": "<consent._id>",
    "piiPrincipalId": "<consent.user>"
  },
  "piiProcessing": {
    "privacyNotice": "<consent.privacyNotice>",
    "language": "en",
    "purposes": [
      {
        "purpose": "<purpose.purpose>",
        "purposeType": "<purpose.purposeType>",
        "lawfulBasis": "consent",
        "piiInformation": "<purpose.piiInformation>",
        "piiControllers": ["<provider.selfDescriptionURL>", "<consumer.selfDescriptionURL>"],
        "collectionMethod": "<purpose.collectionMethod>",
        "processingMethod": "<purpose.processingMethod>",
        "storageLocation": "<consent.storageLocations>",
        "retentionPeriod": "<consent.retentionPeriod>",
        "processingLocations": "<consent.processingLocations>",
        "geographicRestrictions": "<consent.geographicRestrictions>",
        "services": "<consent.services>",
        "jurisdiction": "<consent.jurisdiction>",
        "recipientThirdParties": "<consent.recipientThirdParties>",
        "withdrawalMethod": "<env.WITHDRAWAL_METHOD>",
        "privacyRights": "<env.PRIVACY_RIGHTS (comma-separated)>",
        "codeOfConduct": "<env.CODE_OF_CONDUCT>",
        "impactAssessment": "<env.IMPACT_ASSESSMENT>",
        "authorityParty": "<env.AUTHORITY_PARTY>"
      }
    ]
  },
  "event": "<consent.event (passed through)>",
  "partyIdentification": [
    {
      "partyId": "<consumer.selfDescriptionURL>",
      "partyAddress": "<consumerSelfDescription.legalPerson.legalAddress>",
      "partyName": "<consumerSelfDescription.legalName>",
      "partyContact": "<consumerSelfDescription.legalPerson.subOrganization>",
      "partyType": "consumer"
    },
    {
      "partyId": "<provider.selfDescriptionURL>",
      "partyAddress": "<providerSelfDescription.legalPerson.legalAddress>",
      "partyName": "<providerSelfDescription.legalName>",
      "partyContact": "<providerSelfDescription.legalPerson.subOrganization>",
      "partyType": "provider"
    }
  ]
}
```

**Environment-sourced fields:** Five fields in the receipt come from environment variables rather than the consent document:

| Receipt Field | Environment Variable |
|---------------|---------------------|
| `withdrawalMethod` | `WITHDRAWAL_METHOD` |
| `privacyRights` | `PRIVACY_RIGHTS` (comma-separated, split into array) |
| `codeOfConduct` | `CODE_OF_CONDUCT` |
| `impactAssessment` | `IMPACT_ASSESSMENT` |
| `authorityParty` | `AUTHORITY_PARTY` |

### 11.2 Kantara Consent Receipt (KI-CR-v1.1.0)

A separate utility in `src/libs/consent-generator/` generates Kantara-format receipts. This is a **standalone utility** that is NOT used by the main consent query endpoints.

**Generated fields:**

| Field | Source |
|-------|--------|
| `version` | Static: `"KI-CR-v1.1.0"` |
| `consentTimestamp` | `Date.now()` (Unix milliseconds) |
| `consentReceiptID` | `crypto.randomUUID()` |
| `jurisdiction` | From input options |
| `collectionMethod` | From input options |
| `publicKey` | From input options (optional) |
| `language` | From input options (optional) |
| `piiPrincipalId` | From input options |
| `piiControllers[]` | Array of `{ piiController, contact, address, email, phone }` |
| `policyUrl` | From input options |
| `services[]` | Array of `{ service, purposes[] }` where each purpose has `serviceOffering`, `resource`, `purpose`, `purposeCategory[]`, `consentType`, `piiCategory[]`, `primaryPurpose`, `termination`, `thirdPartyDisclosure`, `thirdPartyName?` |
| `sensitive` | From input options |
| `spiCat[]` | From input options (Sensitive Personal Information categories) |

The generator class provides a `generate(options)` method and a `dummy()` static method for test data.

---

## 12. Consent Querying

### 12.1 List User's Consents

**Endpoint:** `GET /consents/me` `[UserJWT]`
**Endpoint:** `GET /consents/participants/:userId/` `[ParticipantJWT]`

The second endpoint uses the `setUserIdForParticipant` middleware to resolve a participant-scoped `userId` (UserIdentifier) to the actual User.

**Query parameters:**

| Param | Type | Default | Description |
|-------|------|---------|-------------|
| `limit` | String | `"10"` | Number of results per page |
| `page` | String | `"1"` | Page number (1-indexed) |
| `receipt` | String | `"false"` | When `"true"`, returns raw consent documents; when `"false"`, returns ISO/IEC 29184 receipt format |
| `all` | String | `"false"` | When `"true"`, includes superseded consents (those with a `child`); when `"false"`, only returns latest in each chain |

**Logic:**

1. Build a MongoDB filter:
   - If `all` is NOT `"true"`: add `{ child: { $exists: false } }` to exclude superseded consents.
   - For User JWT auth: filter by `user: req.user.id`.
   - For User Key auth: filter by `$or: [{ consumerUserIdentifier }, { providerUserIdentifier }]` matching the identifier.
2. Query with `.skip((page - 1) * limit).limit(limit)` for pagination.
3. Count total documents for pagination metadata.
4. Convert each consent to a consent receipt via `consentToConsentReceipt()`.

**Response:**

```json
{
  "consents": [ /* consent receipts or raw consents */ ],
  "totalCount": 42,
  "totalPages": 5
}
```

> **Warning: Known Bug**
> `totalCount` is computed with `Consent.countDocuments()` with no filter, counting ALL consents in the entire collection rather than just the user's filtered consents.

### 12.2 Get Single User Consent

**Endpoint:** `GET /consents/me/:id` `[UserJWT]`
**Endpoint:** `GET /consents/participants/:userId/:id` `[ParticipantJWT]`

Returns a single consent by ID for the authenticated user, as a consent receipt.

**Response:** `200` with consent receipt, or `404` if not found.

---

## 13. PDI Iframe Redirect

**Endpoint:** `GET /consents/pdi/iframe` `[ParticipantJWT]`

Redirects the client to the Personal Data Intermediary (PDI) frontend for the user to interact with a consent form.

### 13.1 Request

**Query parameters:**

| Param | Type | Description |
|-------|------|-------------|
| `privacyNoticeId` | String | Privacy notice to display |
| `userIdentifier` | String | UserIdentifier ID |

### 13.2 Logic

1. Check that `PDI_ENDPOINT` environment variable is configured. Return `400` if not.
2. Build redirect URL: `{PDI_ENDPOINT}/consents/{privacyNoticeId}?userIdentifier={userIdentifier}`
3. If an existing consent is found for the privacy notice + user identifier combination, append `&consentId={consentId}` to the redirect URL.
4. Redirect the client (HTTP 302) to the constructed URL.

### 13.3 Response

- **302** -- Redirect to PDI frontend
- **400** -- `PDI_ENDPOINT` not configured

---

## 14. Available Exchanges

These endpoints return the list of data exchange opportunities available to a user or participant, derived from bilateral and ecosystem contracts.

### 14.1 User's Available Exchanges

**Endpoint:** `GET /consents/exchanges/user` `[UserJWT]`

**Query parameters:**

| Param | Type | Description |
|-------|------|-------------|
| `participantId` | String | Participant ID to scope the exchange lookup |

**Logic:**

1. Load the Participant document by `participantId`.
2. Call `getAvailableExchangesForParticipant(participant.selfDescriptionURL)` which fetches bilateral and ecosystem contracts from the Contract Service.

**Response:**

```json
{
  "participant": {
    "selfDescription": "<participant.selfDescriptionURL>",
    "base64SelfDescription": "<base64 encoded>"
  },
  "exchanges": [ /* contract objects */ ]
}
```

### 14.2 Participant's Available Exchanges

**Endpoint:** `GET /consents/exchanges/:as` `[ParticipantJWT]`

**Path parameters:**

| Param | Type | Description |
|-------|------|-------------|
| `as` | String | Role: `"provider"` or `"consumer"` |

Uses the session's `userParticipant.id` to look up the participant.

**Response:** Same format as above. Returns `400` if `as` parameter is missing or not `"provider"`/`"consumer"`.

---

## 15. Token Management

Token management supports the data exchange verification protocol, where providers generate tokens that consumers must validate.

### 15.1 Attach Token to Consent

**Endpoint:** `POST /consents/:consentId/token` `[ParticipantJWT]`

**Request body:**

| Field | Type | Description |
|-------|------|-------------|
| `token` | String | Provider-generated exchange token |
| `providerDataExchangeId` | String | Provider's exchange transaction ID |

**Logic:**

1. Find and populate the consent (with both participants and user identifiers, excluding `clientID`/`clientSecret`).
2. Update the consent's `token` field.
3. Encrypt the updated consent payload (with `providerDataExchangeId` included, see [Consent Encryption](#16-consent-encryption)).
4. POST to the consumer's `consentImport` endpoint: `{consent.dataConsumer.endpoints.consentImport}` with body `{ dataProviderEndpoint, signedConsent, encrypted }`.

**Response:** `200` with success message.

### 15.2 Validate Token

**Endpoint:** `POST /consents/:consentId/validate` `[ParticipantJWT]`

**Request body:**

| Field | Type | Description |
|-------|------|-------------|
| `token` | String | Token to validate |

**Logic:** Finds consent by ID, compares `token` from request body against `consent.token`.

**Response:**
- **200** `{ verified: true }` -- Token matches
- **400** `{ verified: false }` -- Token does not match
- **404** -- Consent not found

---

## 16. Consent Encryption

Consent payloads are encrypted before being transmitted to Data Space Connectors during data exchanges. The encryption uses a hybrid AES + RSA scheme.

### 16.1 Encryption Scheme: AES-256-CBC + RSA

**Encryption process (`encryptPayloadAndKey`):**

1. **Generate IV:** A random 16-byte initialization vector is generated using `crypto.randomBytes(16)`.
2. **AES-256-CBC encryption:**
   - The consent payload is JSON-serialized.
   - A pre-loaded AES-256 key (`src/config/keys/AESKey.pem`) is used with the IV to create an AES-256-CBC cipher.
   - The payload is encrypted, producing the ciphertext.
3. **RSA key encryption:**
   - The AES key is encrypted using the RSA private key (`src/config/keys/consentSignature.pem`) with PKCS1 padding.
   - This is actually RSA **signing** (`crypto.privateEncrypt`), not encryption -- the recipient can decrypt with the public key, which proves authenticity of the sender.
4. **Output format:**
   - `signedConsent`: String in format `"<hex IV>:<hex ciphertext>"`
   - `encrypted`: Buffer containing the RSA-encrypted AES key

### 16.2 Decryption (Recipient Side)

To decrypt the consent payload, the recipient must:

1. Obtain the Consent Manager's RSA **public key** (available via `GET /participants/consent-signature`).
2. Decrypt the `encrypted` field using `crypto.publicDecrypt` with the RSA public key to recover the AES key.
3. Split the `signedConsent` string by `:` to extract the IV (first part) and ciphertext (second part).
4. Create an AES-256-CBC decipher with the recovered AES key and IV.
5. Decrypt the ciphertext to obtain the original consent JSON payload.

### 16.3 Key Management

**Key files (in `src/config/keys/`):**

| File | Type | Generation Command |
|------|------|--------------------|
| `consentSignature.pem` | RSA private key | `pnpm run generatePrivateKey` |
| *(derived)* | RSA public key | `pnpm run generatePublicKey` |
| `AESKey.pem` | AES-256 key | `pnpm run generateAES` |

**Key loading:** Both keys are loaded synchronously at module import time using `readFileSync`. If either file does not exist, the entire consent controller module will fail to load, bringing down all consent routes.

### 16.4 Usage in Data Exchange Protocol

The encryption is used at two points during a data exchange:

1. **Consent export (to provider):** When triggering a data exchange, the consent payload is encrypted and sent to the provider's `consentExport` endpoint via POST.
2. **Consent import (to consumer):** When the provider calls back with a token, the updated consent (including token and `providerDataExchangeId`) is encrypted and forwarded to the consumer's `consentImport` endpoint.

---

## 17. Endpoint Reference

Complete list of all consent management endpoints:

| # | Method | Path | Auth | Controller Function | Description |
|---|--------|------|------|--------------------|----|
| 1 | `GET` | `/consents/emailverification` | None | `giveConsentOnEmailValidation` | Email verification callback for consent grant |
| 2 | `GET` | `/consents/me` | UserJWT | `getUserConsents` | List authenticated user's consents (paginated) |
| 3 | `GET` | `/consents/me/:id` | UserJWT | `getUserConsentById` | Get a single consent by ID |
| 4 | `GET` | `/consents/exchanges/user` | UserJWT | `getUserAvailableExchanges` | List available exchanges for user at a participant |
| 5 | `GET` | `/consents/exchanges/:as` | ParticipantJWT | `getAvailableExchanges` | List available exchanges for authenticated participant |
| 6 | `GET` | `/consents/privacy-notices/:privacyNoticeId` | UserKey | `getPrivacyNoticeById` | Get populated privacy notice by ID |
| 7 | `GET` | `/consents/privacy-notices/:privacyNoticeId/user` | UserJWT | `getPrivacyNoticeById` | Get populated privacy notice by ID (JWT auth) |
| 8 | `GET` | `/consents/participants/:userId/` | ParticipantJWT | `getUserConsents` | List consents for a participant-scoped user |
| 9 | `GET` | `/consents/participants/:userId/:id` | ParticipantJWT | `getUserConsentById` | Get single consent for a participant-scoped user |
| 10 | `GET` | `/consents/:userId/:providerId/:consumerId` | UserKey | `getPrivacyNotices` | Fetch privacy notices between provider and consumer |
| 11 | `GET` | `/consents/:userId/:providerURI/:consumerURI/:contractURI` | UserJWT | `getPrivacyNoticesByContract` | Fetch privacy notices for a specific contract |
| 12 | `POST` | `/consents` | UserKey | `giveConsent` | Create consent (participant-initiated) |
| 13 | `POST` | `/consents/user` | UserJWT | `giveConsentUser` | Create consent (user-initiated) |
| 14 | `DELETE` | `/consents/:id` | UserJWT | `revokeConsent` | Revoke a granted consent |
| 15 | `POST` | `/consents/:consentId/data-exchange` | UserJWT | `triggerDataExchange` | Trigger data exchange for a consent |
| 16 | `POST` | `/consents/:consentId/resume` | ParticipantJWT | `resumeConsent` | Resume a pending/draft consent |
| 17 | `POST` | `/consents/:consentId/token` | ParticipantJWT | `attachTokenToConsent` | Attach provider token to consent |
| 18 | `POST` | `/consents/:consentId/validate` | ParticipantJWT | `verifyToken` | Validate a token against consent |
| 19 | `POST` | `/consents/:consentId/refuse` | UserJWT | `refuseConsent` | Refuse an active consent |
| 20 | `POST` | `/consents/:consentId/re-confirm` | UserJWT | `reConfirmConsent` | Re-confirm an existing consent |
| 21 | `POST` | `/consents/:consentId/terminate` | UserJWT | `terminateConsent` | Terminate an active consent |
| 22 | `GET` | `/consents/pdi/iframe` | ParticipantJWT | `redirectPDI` | Redirect to PDI frontend |

---

## 18. Known Bugs

The following bugs exist in the original implementation and should be corrected in a reimplementation:

### 18.1 `giveConsentUser` Refused-But-Granted Bug

**Location:** `src/controllers/consentsController.ts` (lines ~940-962)

**Description:** When `event="refused"` in the `giveConsentUser` function, the consent is created with `status: "granted"` and `consented: true` instead of `status: "refused"` and `consented: false`.

**Root cause:** Copy-paste error from the `event="given"` branch. The event array correctly records `consentEvent.refused`, but the `status` and `consented` fields use the wrong values.

**Comparison:** The `giveConsent` function (User Key flow) handles this correctly, creating the consent with `status: "refused"` and `consented: false` when `event="refused"`.

### 18.2 `giveConsentUser` Parent Update Uses Wrong Field

**Location:** `src/controllers/consentsController.ts` (lines ~996-998)

**Description:** The parent consent update uses `$push: { children: newConsent._id }` (plural "children") instead of `{ child: newConsent._id }` (singular). The Consent schema only defines a `child` field (singular ObjectId ref), so this `$push` creates an unschema'd `children` array field and fails to properly link the parent to the child.

**Comparison:** The `giveConsent` function correctly uses `{ child: newConsent._id }`.

### 18.3 `getUserConsents` Global Count Bug

**Location:** `src/controllers/consentsController.ts`

**Description:** The `totalCount` in the paginated response is computed with `Consent.countDocuments()` with no filter, counting ALL consents in the entire collection rather than just the filtered subset for the authenticated user.

**Impact:** The `totalCount` and `totalPages` values in the response are incorrect for any user who doesn't own every consent in the system.

### 18.4 Missing `await` on `consent.save()` in `refuseConsent` and `terminateConsent`

**Location:** `src/controllers/consentsController.ts`

**Description:** Both `refuseConsent` and `terminateConsent` call `consent.save()` without `await`, meaning the HTTP response is sent before the database write completes. If the save fails, the error is silently swallowed.

### 18.5 `Consent.toReceipt()` References Non-Existent Field

**Location:** `src/models/Consent/Consent.model.ts`

**Description:** The `toReceipt()` instance method references `this.json`, but no `json` field exists on the Consent schema. This method would throw a runtime error if called. (Note: the main query endpoints use `consentToConsentReceipt()` from `src/utils/consentReceipt.ts` instead, so this bug has no user-visible impact.)

### 18.6 Email Verification Endpoint Has No Authentication

**Location:** `GET /consents/emailverification`

**Description:** This endpoint is publicly accessible with no authentication middleware. All consent parameters (participant IDs, user identifiers, privacy notice ID) are passed as plain query string parameters. A malicious actor could craft a URL to create a consent without the user's involvement.

**Recommendation:** Use HMAC-signed tokens or one-time-use verification codes in the email link instead of passing raw IDs.

### 18.7 Key Files Loaded at Module Import

**Location:** Top of `src/controllers/consentsController.ts`

**Description:** Both `consentSignature.pem` and `AESKey.pem` are loaded synchronously via `readFileSync` at module import time. If either file does not exist, the entire consent controller module fails to load, taking down all 22 consent routes. There is no graceful fallback or deferred loading.
