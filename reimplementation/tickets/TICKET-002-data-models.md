# TICKET-002: Data Models

## Metadata

| Field | Value |
|-------|-------|
| **Ticket ID** | TICKET-002 |
| **Title** | Data Models |
| **Priority** | Critical |
| **Complexity** | L (Large) |
| **Dependencies** | TICKET-001 |
| **Related User Stories** | US-CM-024, US-UM-001, US-PM-001, US-CM-001 |

## Description

Implement all five Mongoose models that form the data layer of the Consent Manager: Consent, User, Participant, UserIdentifier, and PrivacyNotice. Each model must include its schema definition, TypeScript interfaces, instance methods, indexes, and validation rules.

## Technical Details

### Model 1: Consent (`src/models/Consent/`)

**Collection:** `consents`

**Schema Fields:**

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `identifier` | String | No | — | External consent identifier |
| `contract` | String | No | — | Contract reference URI |
| `privacyNotice` | ObjectId (ref: PrivacyNotice) | No | — | Associated privacy notice |
| `parent` | ObjectId (ref: Consent) | No | — | Parent consent (for re-grants) |
| `child` | ObjectId (ref: Consent) | No | — | Child consent (re-granted from this) |
| `user` | ObjectId (ref: User) | No | — | Consenting user |
| `dataProvider` | Mixed | No | — | Provider participant info |
| `dataConsumer` | Mixed | No | — | Consumer participant info |
| `providerUserIdentifier` | ObjectId (ref: UserIdentifier) | No | — | Provider-side user identifier |
| `consumerUserIdentifier` | ObjectId (ref: UserIdentifier) | No | — | Consumer-side user identifier |
| `consented` | Boolean | No | `false` | Whether consent is currently active |
| `status` | String (enum) | No | `"pending"` | Lifecycle state |
| `purposes` | Array of Mixed | No | `[]` | Processing purposes |
| `data` | Array of Mixed | No | `[]` | Data categories |
| `recipients` | Array of Mixed | No | `[]` | Data recipients |
| `piiPrincipalRights` | Array of String | No | — | Rights of the data subject |
| `processingLocations` | Array of String | No | — | Where data is processed |
| `storageLocations` | Array of String | No | — | Where data is stored |
| `retentionPeriod` | String | No | — | How long data is retained |
| `withdrawalMethod` | String | No | — | How to withdraw consent |
| `geographicRestrictions` | String | No | — | Geographic constraints |
| `jurisdiction` | String | No | — | Legal jurisdiction |
| `recipientThirdParties` | String | No | — | Third-party recipients |
| `event` | Array of Object | No | `[]` | Lifecycle event log |
| `services` | Mixed | No | — | Service details |
| `token` | String | No | — | Data exchange token |
| `jsonld` | String | No | — | JSON-LD self-description |
| `schema_version` | String | No | `"1"` | Schema version |

**Status Enum Values:** `pending`, `draft`, `granted`, `revoked`, `expired`, `terminated`, `refused`

**Event Object Structure:**
```typescript
{
  eventTime: string;       // ISO datetime
  validityDuration: string;
  eventType: string;       // "explicit"
  eventState: string;      // "consent given" | "consent revoked" | ...
}
```

**Instance Methods:**
- `isValid(): boolean` — Checks if consent is in a valid active state
- `toReceipt(): object` — Transforms consent to ISO/IEC 29184 receipt format
- `getParticipants(): { dataProvider, dataConsumer }` — Returns populated participant data
- `getParticipantsIDs(): { dataProviderId, dataConsumerId }` — Returns participant ObjectIds

**Indexes:**
- `{ user: 1, status: 1 }` — For user consent queries
- `{ dataProvider: 1 }` — For provider-scoped queries
- `{ dataConsumer: 1 }` — For consumer-scoped queries
- `{ privacyNotice: 1 }` — For privacy notice lookups

**Options:** `{ timestamps: true }` (adds `createdAt`, `updatedAt`)

### Model 2: User (`src/models/User/`)

**Collection:** `users`

**Schema Fields:**

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `firstName` | String | Yes | — | User's first name |
| `lastName` | String | Yes | — | User's last name |
| `email` | String | Yes | — | User's email (should be unique) |
| `password` | String | Yes | — | Bcrypt-hashed password |
| `identifiers` | Array of ObjectId (ref: UserIdentifier) | No | `[]` | Linked identifiers |
| `oauth` | Object | No | — | OAuth data |
| `oauth.scopes` | String | No | — | OAuth scopes |
| `oauth.refreshToken` | String | No | — | OAuth refresh token |
| `jsonld` | String | No | — | JSON-LD self-description |
| `schema_version` | String | No | `"1"` | Schema version |

**Pre-save Hook:** Hash password with bcrypt (salt rounds: 10) before save, only if password is modified.

**Instance Methods:**
- `validatePassword(candidatePassword: string): Promise<boolean>` — Compare candidate against stored hash

**Indexes:**
- `{ email: 1 }` — Unique index (fixing original bug 17.9)

**Options:** `{ timestamps: true }`

### Model 3: Participant (`src/models/Participant/`)

**Collection:** `participants`

**Schema Fields:**

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `did` | String | No | — | Decentralized identifier |
| `legalName` | String | Yes | — | Legal entity name |
| `selfDescriptionURL` | String | No | — | Self-description URL |
| `email` | String | No | — | Contact email |
| `legalPerson` | Object | No | — | Legal entity details |
| `legalPerson.registrationNumber` | String | No | — | Registration number |
| `legalPerson.headquartersAddress` | String | No | — | HQ address |
| `legalPerson.legalAddress` | String | No | — | Legal address |
| `legalPerson.parentOrganization` | String | No | — | Parent org |
| `legalPerson.subOrganization` | String | No | — | Sub org |
| `clientID` | String | Yes | — | Authentication client ID |
| `clientSecret` | String | Yes | — | Authentication secret (hash in reimplementation) |
| `dataspaceEndpoint` | String | No | — | Dataspace connector endpoint |
| `endpoints` | Object | No | — | DSC endpoint URLs |
| `endpoints.dataExport` | String | No | — | Data export endpoint |
| `endpoints.dataImport` | String | No | — | Data import endpoint |
| `endpoints.consentExport` | String | No | — | Consent export endpoint |
| `endpoints.consentImport` | String | No | — | Consent import endpoint |
| `jsonld` | String | No | — | JSON-LD self-description |
| `schema_version` | String | No | `"1"` | Schema version |

**Indexes:**
- `{ clientID: 1 }` — Unique index

**Options:** `{ timestamps: true }`

### Model 4: UserIdentifier (`src/models/UserIdentifier/`)

**Collection:** `useridentifiers`

**Schema Fields:**

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `attachedParticipant` | ObjectId (ref: Participant) | Yes | — | Owning participant |
| `email` | String | Yes | — | User's email at this participant |
| `identifier` | String | No | — | Participant-specific user ID |
| `url` | String | No | — | Participant self-description URL |
| `user` | ObjectId (ref: User) | No | — | Linked User (populate during federation) |
| `jsonld` | String | No | — | JSON-LD self-description |
| `schema_version` | String | No | `"1"` | Schema version |

**Indexes:**
- `{ attachedParticipant: 1, email: 1 }` — Compound index for federation lookups
- `{ user: 1 }` — For user-scoped queries

**Options:** `{ timestamps: true }`

### Model 5: PrivacyNotice (`src/models/PrivacyNotice/`)

**Collection:** `privacynotices`

**Schema Fields:**

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `contract` | String | No | — | Contract reference URI |
| `title` | String | No | — | Privacy notice title |
| `lastUpdated` | Date | No | — | Last update timestamp |
| `dataProvider` | ObjectId (ref: Participant) | No | — | Provider participant |
| `archivedAt` | Date | No | — | When archived (null if active) |
| `controllerDetails` | Object | No | — | Data controller info |
| `controllerDetails.name` | String | No | — | Controller name |
| `controllerDetails.contact` | String | No | — | Controller contact |
| `controllerDetails.representative` | String | No | — | Controller representative |
| `controllerDetails.dpo` | String | No | — | Data Protection Officer |
| `dataProcessings` | Array of Mixed | No | — | Processing activities |
| `purposes` | Array of Mixed | No | `[]` | Processing purposes |
| `categoriesOfData` | Array of Mixed | No | `[]` | Data categories |
| `data` | Array of Mixed | No | `[]` | Specific data items |
| `recipients` | Array of Mixed | No | `[]` | Data recipients |
| `internationalTransfers` | Object | No | — | Cross-border transfer info |
| `internationalTransfers.countries` | Array of String | No | — | Destination countries |
| `internationalTransfers.safeguards` | String | No | — | Transfer safeguards |
| `retentionPeriod` | String | No | — | Data retention period |
| `piiPrincipalRights` | Array of String | No | — | Data subject rights |
| `withdrawalOfConsent` | String | No | — | Consent withdrawal method |
| `complaintRights` | String | No | — | Complaint rights info |
| `provisionRequirements` | String | No | — | Data provision requirements |
| `automatedDecisionMaking` | String | No | — | Automated decision info |
| `jsonld` | String | No | — | JSON-LD self-description |
| `schema_version` | String | No | `"1"` | Schema version |

**Indexes:**
- `{ contract: 1, dataProvider: 1 }` — For contract-scoped lookups
- `{ archivedAt: 1 }` — For filtering active vs. archived notices

**Options:** `{ timestamps: true }`

### TypeScript Interfaces

For each model, create:
- `I<Model>` — Document interface (extends `mongoose.Document`)
- `I<Model>Model` — Model interface (extends `mongoose.Model<I<Model>>`)
- Export from `src/types/` for use across the codebase

### Files to Create

```
src/models/
  Consent/
    Consent.model.ts        # Schema, model, methods
    index.ts                # Re-export
  User/
    User.model.ts           # Schema, model, pre-save hook, methods
    index.ts
  Participant/
    Participant.model.ts    # Schema, model
    index.ts
  UserIdentifier/
    UserIdentifier.model.ts # Schema, model
    index.ts
  PrivacyNotice/
    PrivacyNotice.model.ts  # Schema, model
    index.ts
src/types/
  models.ts                 # All model interfaces
```

## Acceptance Criteria

1. [ ] All 5 Mongoose models compile without TypeScript errors
2. [ ] Each model has a TypeScript interface exported from `src/types/models.ts`
3. [ ] User model has a bcrypt pre-save hook that hashes passwords
4. [ ] User model has a `validatePassword()` instance method
5. [ ] Consent model has `isValid()`, `toReceipt()`, `getParticipants()`, and `getParticipantsIDs()` instance methods
6. [ ] Consent status enum enforces only valid values: `pending`, `draft`, `granted`, `revoked`, `expired`, `terminated`, `refused`
7. [ ] All models have `timestamps: true` enabled
8. [ ] User email has a unique index
9. [ ] Participant clientID has a unique index
10. [ ] Unit tests verify schema validation, pre-save hooks, and instance methods
11. [ ] `toReceipt()` correctly references schema fields (not non-existent `this.json`)

## Known Issues to Address

1. **`toReceipt()` references non-existent field** (original bug 18.5): The original `toReceipt()` method references `this.json` which does not exist. Implement the receipt transformation using actual schema fields.
2. **No unique index on User email** (original bug 17.9): Add a unique index on `User.email` to prevent duplicate registrations.
3. **`user` field on UserIdentifier unused** (original bug 17.10): The `user` field exists but is never set in the original. In the reimplementation, actively populate this field during user federation (TICKET-006).
4. **Participant `clientSecret` stored in plaintext** (original bug 05-PM-12.3): Hash `clientSecret` with bcrypt, similar to User passwords.
5. **`validatePassword` declared but not implemented on Participant** (original bug 05-PM-12.7): Implement the method properly if `clientSecret` is hashed, or remove the declaration.
6. **Parent/child consent uses wrong field name** (original bug 18.2): The schema defines `child` (singular ObjectId), but code pushes to `children` (plural array). Decide on one approach and implement consistently — recommend using `child` (singular) since re-grant creates exactly one child.
