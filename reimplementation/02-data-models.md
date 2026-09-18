# Data Models

This document provides complete field-level documentation for all five Mongoose data models used by the Consent Manager. Each model includes every field with its type, constraints, defaults, and references, as well as instance methods, hooks, virtuals, and indexes.

All models share two common fields inherited from the `AllSchemas` interface:
- `schema_version` (String) - Schema version for forward compatibility
- `jsonld` (String) - JSON-LD self-description of the entity

All models use `{ timestamps: true }`, which auto-generates `createdAt` and `updatedAt` fields (both `Date` type).

---

## 1. Consent Model

**File:** `src/models/Consent/Consent.model.ts`  
**Collection:** `consents`  
**Purpose:** Represents a user's consent for a specific data exchange between a data provider and a data consumer, linked to a contract and privacy notice. This is the central entity of the system.

### 1.1 Fields

#### Core Identity & Linking

| Field | Type | Required | Default | Ref | Description |
|-------|------|----------|---------|-----|-------------|
| `identifier` | String | No | — | — | Deprecated. Legacy identifier field. |
| `contract` | String | **Yes** | — | — | ID of the contract (from the Contract Service) that this consent is based on. Stored as a string (not ObjectId) because contracts are external. |
| `privacyNotice` | String | No | `""` | — | ID of the associated PrivacyNotice document (stored as string). |
| `parent` | ObjectId | No | — | `Consent` | Reference to the parent consent when this consent is a re-grant (child replaces parent). |
| `child` | ObjectId | No | — | `Consent` | Reference to the child consent that replaced this one upon re-grant. |

#### Participants & Users

| Field | Type | Required | Default | Ref | Description |
|-------|------|----------|---------|-----|-------------|
| `user` | ObjectId | Conditional | — | `User` | The end user who granted consent. **Required when `status` is `granted`, `revoked`, or `expired`** (enforced by custom validator). |
| `dataProvider` | ObjectId | **Yes** | — | `Participant` | The participant providing data. |
| `dataConsumer` | ObjectId | **Yes** | — | `Participant` | The participant consuming data. |
| `providerUserIdentifier` | ObjectId | **Yes** | — | `UserIdentifier` | The user's identity as known to the data provider. |
| `consumerUserIdentifier` | ObjectId | No (`required: false`) | — | `UserIdentifier` | The user's identity as known to the data consumer. May be absent if the user has no account with the consumer. |

#### Consent State

| Field | Type | Required | Default | Ref | Description |
|-------|------|----------|---------|-----|-------------|
| `consented` | Boolean | **Yes** | — | — | Whether the user has actively consented. `true` when granted, `false` when revoked/refused/terminated. |
| `status` | String (enum) | No | — | — | Current lifecycle state. See [Consent States](#13-consent-states) below. |

**Status enum values:** `pending`, `draft`, `granted`, `revoked`, `expired`, `terminated`, `refused`

#### Data & Purposes

| Field | Type | Required | Default | Ref | Description |
|-------|------|----------|---------|-----|-------------|
| `purposes` | Array of subdocuments | No | — | — | Data processing purposes covered by this consent. See [Purposes subdocument](#purposes-subdocument) below. |
| `data` | Array of subdocuments | No | — | — | Data resources covered by this consent. See [Data subdocument](#data-subdocument) below. |
| `recipients` | [String] | Yes (per item) | — | — | List of data recipient identifiers. |

##### Purposes Subdocument

Each element in the `purposes` array has:

| Field | Type | Description |
|-------|------|-------------|
| `_id` | String | Purpose identifier |
| `purpose` | String | Human-readable purpose description |
| `resource` | String | Resource this purpose applies to |
| `serviceOffering` | String | Service offering ID associated with this purpose |
| `purposeType` | String | Type/category of the purpose |
| `piiInformation` | Array of subdocuments | PII details. See below. |
| `collectionMethod` | [String] | How data is collected |
| `processingMethod` | [String] | How data is processed |

**PII Information subdocument** (within `purposes[].piiInformation`):

| Field | Type | Description |
|-------|------|-------------|
| `piiType` | String | Type of personally identifiable information |
| `piiAttributeId` | String | Attribute identifier for the PII |
| `piiOptional` | String | Whether this PII is optional |
| `sensitivePiiCategory` | String | Sensitive PII classification |
| `specialPiiCategory` | String | Special category PII classification |

##### Data Subdocument

Each element in the `data` array has:

| Field | Type | Description |
|-------|------|-------------|
| `_id` | String | Data resource identifier |
| `resource` | String | Resource URL or identifier |
| `serviceOffering` | String | Service offering ID |

#### Privacy & Compliance

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `piiPrincipalRights` | [String] | No | — | Rights of the PII principal (data subject rights). |
| `processingLocations` | [String] | No | — | Locations where data processing occurs. |
| `storageLocations` | [String] | No | — | Locations where data is stored. |
| `retentionPeriod` | String | No | — | How long data is retained. |
| `withdrawalMethod` | String | No | — | Method for withdrawing consent. |
| `geographicRestrictions` | [String] | No | — | Geographic restrictions on data processing. |
| `jurisdiction` | String | No | — | Legal jurisdiction governing this consent. |

#### Third-Party Recipients

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `recipientThirdParties` | Subdocument | No | — | Third-party infrastructure services involved. |

**`recipientThirdParties` subdocument:**

| Field | Type | Description |
|-------|------|-------------|
| `catalogId` | String | Catalog identifier |
| `infrastructureServices` | Array of subdocuments | See below. |

**`infrastructureServices[]` subdocument:**

| Field | Type | Description |
|-------|------|-------------|
| `name` | String | Service name |
| `location` | String | Service location |
| `natureOfDataAccess` | String | How the service accesses data |
| `participant` | String | Participant identifier |
| `serviceOffering` | String | Service offering identifier |
| `params` | ObjectId | Raw ObjectId (no ref) for service parameters |
| `config` | String | Service configuration |

#### Events & Audit Trail

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `event` | Array of subdocuments | No | — | Chronological log of consent lifecycle events. |

**Event subdocument:**

| Field | Type | Description |
|-------|------|-------------|
| `eventTime` | String | ISO timestamp of when the event occurred |
| `validityDuration` | String | Duration for which this event's state is valid |
| `eventType` | String | Type of event (e.g., consent creation, grant, revocation) |
| `eventState` | String | State that resulted from this event |

#### Miscellaneous

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `services` | [String] | No | — | List of services covered by this consent. |
| `token` | String | No | `""` | Provider-attached token for data exchange verification. |
| `jsonld` | String | No | — | JSON-LD representation of the consent receipt. |
| `schema_version` | String | No | `"0.2.0"` | Schema version. |

### 1.2 Custom Validators

Two cross-field validators enforce that `user` is present when the consent reaches certain states:

1. **On the `user` path:** Fails validation if `status` is `granted`, `revoked`, or `expired` and `user` is falsy. Message: *"User is required when status is granted, revoked or expired"*.
2. **On the `status` path:** Mirror of the above validator with the same logic and message.

### 1.3 Consent States

```
                  +---------+
                  | pending |
                  +----+----+
                       |
              grant / resume
                       |
                  +----v----+
                  |  draft  |
                  +----+----+
                       |
              grant / resume
                       |
                  +----v----+
         +------->| granted |<-------+
         |        +--+-+-+--+        |
    re-grant         | | |       re-confirm
         |    revoke | | | terminate
         |           | | |
    +----+---+       | | |     +-----------+
    | granted |<--+  | | +---->| terminated |
    | (child) |   |  | |      +-----------+
    +---------+   |  | |
                  |  | +----------+
                  |  |            |
              +---v--v--+   +----v---+
              | revoked |   | refused |
              +---------+   +--------+
```

**Valid states:**
- **`pending`** - Initial state. Consent created but awaiting user action (e.g., email verification).
- **`draft`** - Consent prepared but not yet granted.
- **`granted`** - User has actively consented. `consented` is `true`.
- **`revoked`** - User withdrew consent. `consented` is `false`.
- **`expired`** - Consent's validity period has passed.
- **`terminated`** - Consent was terminated (e.g., by a participant or system action). `consented` is `false`.
- **`refused`** - User explicitly refused to consent. `consented` is `false`.

### 1.4 Instance Methods

#### `isValid(): boolean`
Returns `true` if `status === "granted"` and `consented === true`. Used to check whether a consent is currently active and can be used for data exchange.

#### `toReceipt(): object`
Parses the `jsonld` field (note: implementation references `this.json`, which may be a bug — likely intended to use `this.jsonld`) and returns the parsed object. Used to extract a structured consent receipt.

#### `getParticipants(): { assigner: string, assignee: string }`
Parses the `jsonld` field, navigates to `permission[0]`, and extracts `assigner` (data provider) and `assignee` (data consumer) URIs. Throws `NotFoundError` if the permission structure is missing.

#### `getParticipantsIDs(): { assignerID: string, assigneeID: string }`
Calls `getParticipants()` and extracts the last URI path segment from each as the participant MongoDB ID. For example, `http://example.com/participants/abc123` yields `abc123`.

### 1.5 Virtuals

#### `jsonData` (getter)
Parses `this.jsonld` as JSON and returns the resulting object. Provides convenient access to the structured consent data without manual parsing.

### 1.6 Indexes

No explicit indexes are defined beyond MongoDB's default `_id` index.

---

## 2. User Model

**File:** `src/models/User/User.model.ts`  
**Collection:** `users`  
**Purpose:** Represents an end user who can grant consent for data exchanges. Users may have accounts with multiple participants, linked via UserIdentifiers.

### 2.1 Fields

| Field | Type | Required | Default | Ref | Description |
|-------|------|----------|---------|-----|-------------|
| `firstName` | String | No | `""` | — | User's first name. |
| `lastName` | String | No | `""` | — | User's last name. |
| `email` | String | **Yes** | `""` | — | User's email address. Primary identifier for cross-participant federation. |
| `password` | String | No | — | — | bcrypt-hashed password. Optional because users may be created via participant registration without a password. |
| `identifiers` | [ObjectId] | No | — | `UserIdentifier` | Array of UserIdentifier references linking this user to various participants. |
| `oauth` | Subdocument | No | — | — | OAuth token storage. |
| `oauth.scopes` | [String] | No | — | — | Granted OAuth scopes. |
| `oauth.refreshToken` | String | No | — | — | OAuth refresh token. |
| `jsonld` | String | No | `""` | — | JSON-LD self-description. |
| `schema_version` | String | **Yes** | `"v0.1.0"` | — | Schema version. |

### 2.2 Instance Methods

#### `validatePassword(password: string): Promise<boolean>`
Compares the provided plaintext password against the stored bcrypt hash using `bcrypt.compare()`. Returns `true` if the password matches, `false` otherwise.

### 2.3 Pre-Save Hook

**Trigger:** `pre("save")`  
**Condition:** Runs only if the document is new (`this.isNew`) or the `password` field has been modified (`this.isModified("password")`).  
**Action:** Hashes the plaintext password using `bcrypt.hash()` with the number of salt rounds specified by the `SALT_ROUNDS` environment variable (parsed as integer, default `10`). Replaces `this.password` with the hash.

### 2.4 Indexes

No explicit indexes beyond `_id`.

---

## 3. Participant Model

**File:** `src/models/Participant/Participant.model.ts`  
**Collection:** `participants`  
**Purpose:** Represents an organization participating in the data space — either as a data provider, data consumer, or both. Participants register with the Consent Manager and configure their Data Space Connector endpoints.

### 3.1 Fields

#### Identity

| Field | Type | Required | Default | Ref | Description |
|-------|------|----------|---------|-----|-------------|
| `did` | String | **Yes** | `""` | — | Decentralized Identifier of the participant. |
| `legalName` | String | **Yes** | — | — | Legal name of the organization. |
| `selfDescriptionURL` | String | No | `""` | — | URL to the participant's self-description document. Used as a unique identifier in contract and consent operations. |
| `email` | String | **Yes** | `""` | — | Contact email for the participant. |

#### Legal Person (Nested Object)

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `legalPerson` | Subdocument | No | — | Legal entity details. |
| `legalPerson.registrationNumber` | String | No | `""` | Business registration number. |
| `legalPerson.headquartersAddress` | Subdocument | No | — | Headquarters address. |
| `legalPerson.headquartersAddress.countryCode` | String | No | `""` | ISO country code. |
| `legalPerson.legalAddress` | Subdocument | No | — | Legal/registered address. |
| `legalPerson.legalAddress.countryCode` | String | No | `""` | ISO country code. |
| `legalPerson.parentOrganization` | [String] | No | — | Parent organization identifiers. |
| `legalPerson.subOrganization` | [String] | No | — | Sub-organization identifiers. |

#### Authentication

| Field | Type | Required | Default | Ref | Description |
|-------|------|----------|---------|-----|-------------|
| `clientID` | String | **Yes** | — | — | Unique client identifier for API authentication. Also used by DSCs to authenticate via the `serviceKey` JWT path. |
| `clientSecret` | String | **Yes** | — | — | Secret for JWT signing verification and DSC login. |

#### DSC Endpoints

| Field | Type | Required | Default | Ref | Description |
|-------|------|----------|---------|-----|-------------|
| `dataspaceEndpoint` | String | No | — | — | Base URL of the participant's Data Space Connector. Used for user registration and authentication. |
| `endpoints` | Subdocument | No | — | — | Specific operation endpoints on the DSC. |
| `endpoints.dataExport` | String | No | `""` | — | Endpoint for data export operations. |
| `endpoints.dataImport` | String | No | `""` | — | Endpoint for data import operations. |
| `endpoints.consentExport` | String | No | `""` | — | Endpoint for receiving encrypted consent for export-side processing. |
| `endpoints.consentImport` | String | No | `""` | — | Endpoint for receiving encrypted consent for import-side processing. |

#### Metadata

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `jsonld` | String | No | `""` | JSON-LD self-description. |
| `schema_version` | String | No | `"v0.1.0"` | Schema version. |

### 3.2 Instance Methods, Hooks, Virtuals, Indexes

None defined.

---

## 4. UserIdentifier Model

**File:** `src/models/UserIdentifier/UserIdentifier.model.ts`  
**Collection:** `useridentifiers`  
**Purpose:** Links a user's identity at a specific participant to their global User account in the Consent Manager. This is the key entity for cross-participant user federation — it maps participant-local identifiers (email, URL, custom identifier) to a single Consent Manager User.

### 4.1 Fields

| Field | Type | Required | Default | Ref | Description |
|-------|------|----------|---------|-----|-------------|
| `attachedParticipant` | ObjectId | **Yes** | — | `Participant` | The participant this identifier belongs to. |
| `email` | String | **Yes** | — | — | User's email address at this participant. Used for cross-participant matching. |
| `identifier` | String | No | — | — | Participant-specific user identifier (e.g., internal user ID in the participant's system). |
| `url` | String | No | — | — | URL-based identifier for the user at this participant. |
| `user` | ObjectId | No | — | `User` | Reference to the global User account. May be `null` initially and linked later via email matching. |
| `jsonld` | String | No | `""` | — | JSON-LD self-description. |
| `schema_version` | String | No | `"v0.1.0"` | — | Schema version. |

### 4.2 Cross-Participant Federation Role

UserIdentifier is central to the user federation mechanism:

1. When a participant registers a user (`POST /users/register`), a UserIdentifier is created linking the participant to the user's email and optional identifier/URL.
2. The system attempts to find an existing User with the same email. If found, the UserIdentifier is linked to that User and added to the User's `identifiers` array.
3. If no existing User is found, a new User is created with the email from the UserIdentifier.
4. When the same user registers with a different participant, a new UserIdentifier is created but linked to the same User (matched by email).
5. This allows consents to reference participant-specific identities (`providerUserIdentifier`, `consumerUserIdentifier`) while maintaining a single User identity across the ecosystem.

### 4.3 Instance Methods, Hooks, Virtuals, Indexes

None defined.

---

## 5. PrivacyNotice Model

**File:** `src/models/PrivacyNotice/PrivacyNotice.model.ts`  
**Collection:** `privacynotices`  
**Purpose:** Stores privacy notices generated from contracts. A privacy notice describes the data processing that will occur under a contract and must be presented to the user before consent is granted. Privacy notices can be archived when they become stale (e.g., contract changes).

### 5.1 Fields

#### Core

| Field | Type | Required | Default | Ref | Description |
|-------|------|----------|---------|-----|-------------|
| `contract` | String | No | — | — | ID of the contract this privacy notice was generated from. |
| `title` | String | No | — | — | Human-readable title of the privacy notice. |
| `lastUpdated` | String | No | — | — | ISO timestamp of last update. |
| `dataProvider` | String | No | — | — | Identifier of the data provider participant. |
| `archivedAt` | String | No | — | — | ISO timestamp when this notice was archived. `null`/absent means the notice is active. |

#### Controller Details (Nested Object)

| Field | Type | Description |
|-------|------|-------------|
| `controllerDetails` | Subdocument | Details about the data controller. |
| `controllerDetails.name` | String | Controller organization name. |
| `controllerDetails.contact` | String | Controller contact information. |
| `controllerDetails.representative` | String | Controller representative. |
| `controllerDetails.dpo` | Subdocument | Data Protection Officer details. |
| `controllerDetails.dpo.name` | String | DPO name. |
| `controllerDetails.dpo.contact` | String | DPO contact information. |

#### Data Processing

| Field | Type | Description |
|-------|------|-------------|
| `dataProcessings` | [Mixed] | Array of data processing activities (flexible schema). |
| `purposes` | [Mixed] | Array of processing purposes (flexible schema). |
| `categoriesOfData` | [String] | Categories of personal data processed. |
| `data` | [Mixed] | Array of specific data elements (flexible schema). |
| `recipients` | [String] | List of data recipients. |

#### Legal & Compliance

| Field | Type | Description |
|-------|------|-------------|
| `internationalTransfers` | Subdocument | International data transfer details. |
| `internationalTransfers.countries` | [String] | Countries data may be transferred to. |
| `internationalTransfers.safeguards` | String | Safeguards for international transfers. |
| `retentionPeriod` | String | How long data will be retained. |
| `piiPrincipalRights` | [String] | Rights of the data subject. |
| `withdrawalOfConsent` | String | How to withdraw consent. |
| `complaintRights` | String | Rights to file complaints. |
| `provisionRequirements` | String | Requirements for data provision. |
| `automatedDecisionMaking` | Subdocument | Automated decision-making details. |
| `automatedDecisionMaking.details` | String | Details of automated decisions. |

#### Metadata

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `jsonld` | String | No | — | JSON-LD self-description. |
| `schema_version` | String | No | `"0.1.0"` | Schema version. |

### 5.2 Archival Lifecycle

Privacy notices follow a simple active/archived lifecycle:
- **Active:** `archivedAt` is `null` or absent. The notice can be used for new consents.
- **Archived:** `archivedAt` contains an ISO timestamp. Archived notices are no longer used for new consents but are preserved for historical reference (existing consents may still reference them).

Archival occurs when the consent controller detects that a contract has changed and the existing privacy notice is stale.

### 5.3 Instance Methods, Hooks, Virtuals, Indexes

None defined.

---

## 6. Entity Relationship Diagram

```
+------------------+       +---------------------+       +------------------+
|      User        |       |   UserIdentifier    |       |   Participant    |
+------------------+       +---------------------+       +------------------+
| _id              |<------| user (optional)     |------>| _id              |
| firstName        |  1:N  | attachedParticipant |  N:1  | did              |
| lastName         |       | email               |       | legalName        |
| email            |       | identifier          |       | legalPerson{}    |
| password (hash)  |       | url                 |       | selfDescriptionURL|
| identifiers[]  --+------>| jsonld              |       | email            |
| oauth{}          |       | schema_version      |       | dataspaceEndpoint|
| jsonld           |       +---------------------+       | endpoints{}      |
| schema_version   |              |    |                  | clientID         |
+------------------+              |    |                  | clientSecret     |
       |                          |    |                  | jsonld           |
       | 1:N                      |    |                  | schema_version   |
       |                          |    |                  +------------------+
       |    +---------------------+    |                        |   |
       |    |                          |                        |   |
       v    v                          v                        v   v
+------------------------------------------------------------------+
|                          Consent                                 |
+------------------------------------------------------------------+
| _id                                                              |
| contract (String, external ID)                                   |
| user -----------------> User._id                                 |
| dataProvider ----------> Participant._id                         |
| dataConsumer ----------> Participant._id                         |
| providerUserIdentifier -> UserIdentifier._id                    |
| consumerUserIdentifier -> UserIdentifier._id                    |
| parent ----------------> Consent._id (self-ref)                 |
| child -----------------> Consent._id (self-ref)                 |
| consented (Boolean)                                              |
| status (enum)                                                    |
| purposes[] { purpose, resource, serviceOffering, piiInfo[] }    |
| data[] { resource, serviceOffering }                             |
| recipients[]                                                     |
| recipientThirdParties { catalogId, infrastructureServices[] }   |
| event[] { eventTime, validityDuration, eventType, eventState }  |
| privacyNotice (String, internal ID)                              |
| token, jsonld, schema_version, ...                               |
+------------------------------------------------------------------+
       |
       | references (by string ID)
       v
+------------------------------------------------------------------+
|                       PrivacyNotice                              |
+------------------------------------------------------------------+
| _id                                                              |
| contract (String, external ID)                                   |
| title                                                            |
| dataProvider (String, participant ID)                            |
| controllerDetails { name, contact, representative, dpo{} }     |
| purposes[] (Mixed)                                               |
| categoriesOfData[]                                               |
| data[] (Mixed)                                                   |
| recipients[]                                                     |
| internationalTransfers { countries[], safeguards }              |
| retentionPeriod                                                  |
| piiPrincipalRights[]                                             |
| withdrawalOfConsent, complaintRights, provisionRequirements     |
| automatedDecisionMaking { details }                              |
| archivedAt                                                       |
| jsonld, schema_version                                           |
+------------------------------------------------------------------+
```

### Relationship Summary

| From | To | Cardinality | Field | Notes |
|------|----|-------------|-------|-------|
| User | UserIdentifier | 1:N | `User.identifiers[]` | User has many identifiers across participants |
| UserIdentifier | User | N:1 | `UserIdentifier.user` | Optional; linked after email matching |
| UserIdentifier | Participant | N:1 | `UserIdentifier.attachedParticipant` | Required; each identifier belongs to one participant |
| Consent | User | N:1 | `Consent.user` | Required when granted/revoked/expired |
| Consent | Participant | N:1 (x2) | `Consent.dataProvider`, `Consent.dataConsumer` | Each consent has exactly one provider and one consumer |
| Consent | UserIdentifier | N:1 (x2) | `Consent.providerUserIdentifier`, `Consent.consumerUserIdentifier` | Links to participant-specific user identities |
| Consent | Consent | 1:1 | `Consent.parent`, `Consent.child` | Self-referential for re-grant chains |
| Consent | PrivacyNotice | N:1 | `Consent.privacyNotice` (string) | References by ID string, not ObjectId ref |
| PrivacyNotice | Contract | N:1 | `PrivacyNotice.contract` (string) | External contract reference by string ID |

### Key Design Decisions

1. **External IDs as strings:** Contract IDs and some participant references are stored as plain strings rather than MongoDB ObjectIds because they refer to entities in external services (Contract Service, Catalog).

2. **Flexible schema for PrivacyNotice:** The `purposes`, `data`, and `dataProcessings` fields use `Mixed` type, allowing the schema to adapt to different contract structures from the Contract Service without migration.

3. **Conditional user requirement on Consent:** The `user` field is only required when the consent reaches certain states (`granted`, `revoked`, `expired`). This allows consent creation to begin without a resolved user (e.g., during the email verification flow) and link the user later.

4. **Dual user identity on Consent:** Both `providerUserIdentifier` and `consumerUserIdentifier` are stored because the same user may be known by different identifiers at the provider and consumer organizations.

5. **Event array as audit trail:** The `event` array on Consent provides a chronological log of all lifecycle events, supporting auditability requirements without a separate audit table.

6. **Password optionality on User:** The `password` field is optional because users can be created through participant registration (where the participant vouches for the user's identity) without the user needing a direct login.
