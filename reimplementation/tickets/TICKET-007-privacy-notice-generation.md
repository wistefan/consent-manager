# TICKET-007: Privacy Notice Generation

## Metadata

| Field | Value |
|-------|-------|
| **Ticket ID** | TICKET-007 |
| **Title** | Privacy Notice Generation |
| **Priority** | High |
| **Complexity** | M (Medium) |
| **Dependencies** | TICKET-002, TICKET-003, TICKET-005 |
| **Related User Stories** | US-CM-001, US-CM-002, US-CM-022 |

## Description

Implement privacy notice generation from contracts fetched from the Contract Service, privacy notice archival, and retrieval with external data population. Privacy notices are the bridge between contracts (managed by the Contract Service) and consents (managed by this service) — they describe what data will be processed, for what purpose, and under what conditions.

## Technical Details

### Endpoints

#### 1. Generate Privacy Notice Between Parties — `POST /consents/privacy-notices`

**Auth:** `verifyParticipantJWT`

**Request Body:**
```json
{
  "contract": "string (required — contract URI)",
  "dataProvider": "string (required — base64-encoded provider self-description URL)",
  "dataConsumer": "string (required — base64-encoded consumer self-description URL)"
}
```

**Flow:**
1. Decode base64 provider and consumer self-description URLs
2. Find the Provider and Consumer Participant documents by `selfDescriptionURL`
3. Fetch the contract from the Contract Service: `GET {CONTRACT_SERVICE_URL}/contracts/{contractId}`
4. Check if an active (non-archived) PrivacyNotice already exists for this contract + provider
5. If one exists, return it (no duplicate creation)
6. If a stale one exists (contract has been updated since), archive it (`archivedAt = new Date()`)
7. Extract privacy notice fields from the contract:
   - `purposes` from contract service offerings
   - `data` / `categoriesOfData` from data resource descriptions
   - `recipients` from contract parties
   - `controllerDetails` from provider participant info
   - `retentionPeriod`, `piiPrincipalRights`, `internationalTransfers` from contract terms
8. Create a new PrivacyNotice document
9. Return the created privacy notice

**Response:** `201 Created` with PrivacyNotice object

#### 2. Generate Privacy Notice by Contract URI — Alternative path using contract URI directly

**Auth:** `verifyParticipantJWT`

Same as above but the contract is identified by its full URI rather than a separate contract ID.

#### 3. Retrieve Privacy Notice by ID — `GET /consents/privacy-notices/:id`

**Auth:** `verifyParticipantJWT` or `verifyUserKey`

**Flow:**
1. Find PrivacyNotice by ID
2. Populate external data:
   - Fetch full contract details from Contract Service
   - Fetch data resource descriptions from service offering endpoints
   - Fetch recipient details
   - Fetch purpose descriptions
3. Return the privacy notice with populated data

**Response:** `200 OK` with populated PrivacyNotice

### Contract Service Integration

**External Service:** Contract Service at `CONTRACT_SERVICE_URL`

**Endpoints Used:**
- `GET /contracts/{contractId}` — Fetch contract details
- `GET /contracts/bilateral` — Fetch bilateral contracts for a participant
- `GET /contracts/ecosystem` — Fetch ecosystem contracts for a participant

**Data Extraction from Contract:**

The contract response contains service offerings with:
- `serviceOffering` — URL to the service offering description
- `participant` — Provider or consumer identifier
- `policies` — Data processing policies including purposes

For each service offering, fetch the full description to extract:
- `dataResources` — List of data categories and items
- `softwareResources` — Related software components

### Archival Logic

When a new privacy notice is generated for a contract that already has one:
1. Set `archivedAt = new Date()` on the existing notice
2. Create a new notice with current data
3. Archived notices remain queryable but are excluded from active lookups

### Files to Create/Modify

```
src/controllers/consentsController.ts  # Privacy notice functions (or separate controller)
src/routes/consent.ts                  # Privacy notice routes
```

### External Data Population

When retrieving a privacy notice, the response should include populated data fetched from external services:

```typescript
interface PopulatedPrivacyNotice {
  // ... all PrivacyNotice fields ...
  populatedContract: object;      // Full contract details
  populatedDataResources: object[]; // Resolved data resource descriptions
  populatedPurposes: object[];     // Resolved purpose descriptions
  populatedRecipients: object[];   // Resolved recipient details
}
```

Use `axios` with error handling for all external service calls. If an external fetch fails, include the error in the response rather than failing the entire request.

## Acceptance Criteria

1. [ ] `POST /consents/privacy-notices` generates a privacy notice from a contract
2. [ ] Duplicate privacy notices are not created for the same contract + provider
3. [ ] Stale privacy notices are archived when the contract has been updated
4. [ ] `GET /consents/privacy-notices/:id` returns the privacy notice with populated external data
5. [ ] External service failures are handled gracefully (logged, partial data returned)
6. [ ] Privacy notice fields are correctly extracted from contract service offerings
7. [ ] Archived notices have `archivedAt` set and are excluded from active lookups
8. [ ] Integration tests mock the Contract Service and verify notice generation

## Known Issues to Address

1. **External data population failures**: The original code does not gracefully handle failures when fetching service offerings or data resources. Implement fallback behavior (return the notice with unpopulated fields rather than failing).
2. **Contract caching**: Consider using axios-cache-interceptor to avoid redundant contract fetches, as the original does.
3. **Base64 URL encoding**: Provider and consumer self-description URLs are base64-encoded in request parameters. Ensure consistent encoding/decoding across all endpoints.
