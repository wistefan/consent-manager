# TICKET-010: Consent Querying

## Metadata

| Field | Value |
|-------|-------|
| **Ticket ID** | TICKET-010 |
| **Title** | Consent Querying |
| **Priority** | High |
| **Complexity** | M (Medium) |
| **Dependencies** | TICKET-008 |
| **Related User Stories** | US-CM-011, US-CM-012, US-CM-013, US-CM-015, US-CM-016, US-CM-019 |

## Description

Implement consent listing, retrieval by ID, receipt format transformation, parent/child inclusion, participant-scoped queries, and available exchange discovery endpoints.

## Technical Details

### Endpoints

#### 1. List My Consents — `GET /consents/me`

**Auth:** `verifyUserJWT`

**Query Parameters:**
- `page` (number, default: 1)
- `limit` (number, default: 20)
- `status` (string, optional — filter by status)
- `receipt` (boolean, optional — return in receipt format)
- `includeChildren` (boolean, optional — include child consents)

**Flow:**
1. Get authenticated user from `req.user`
2. Build query filter: `{ user: req.user._id }`
3. If `status` is provided, add to filter
4. Count matching documents (filtered, not global — fixing original bug)
5. Fetch paginated results with population of `privacyNotice`, `dataProvider`, `dataConsumer`
6. If `receipt=true`, transform each consent using `consent.toReceipt()`
7. If `includeChildren=true`, populate `child` references recursively
8. Return paginated response

**Response:**
```json
{
  "consents": [...],
  "totalCount": 42,
  "page": 1,
  "limit": 20
}
```

#### 2. Get My Consent by ID — `GET /consents/me/:id`

**Auth:** `verifyUserJWT`

**Flow:**
1. Find consent by ID
2. Verify the authenticated user owns this consent
3. Populate related documents
4. If `receipt` query param is set, transform to receipt format
5. If `includeChildren` is set, include child consent chain
6. Return consent

**Response:** `200 OK` with consent object

#### 3. List Consents as Participant — `GET /consents/participants/:userId/`

**Auth:** `verifyParticipantJWT`

**Flow:**
1. Get authenticated participant from `req.participant`
2. Resolve the `userId` parameter to a User
3. Build query filter: consents where the participant is provider or consumer AND user matches
4. Apply pagination
5. Return filtered consents

**Response:** `200 OK` with paginated consents

#### 4. Get Consent as Participant by ID — `GET /consents/participants/:userId/:id`

**Auth:** `verifyParticipantJWT`

**Flow:**
1. Find consent by ID
2. Verify the authenticated participant is the provider or consumer
3. Return consent with populated data

**Response:** `200 OK` with consent object

#### 5. Discover Available Exchanges as User — `GET /consents/exchanges/user`

**Auth:** `verifyUserJWT`

**Flow:**
1. Get authenticated user
2. Find all participants linked to the user via UserIdentifiers
3. For each participant, fetch bilateral contracts from Contract Service:
   - `GET {CONTRACT_SERVICE_URL}/contracts/bilateral?participantId={id}`
4. Aggregate and deduplicate contracts
5. Return available exchanges

**Response:** `200 OK` with list of available exchanges

#### 6. Discover Available Exchanges as Participant — `GET /consents/exchanges/:as`

**Auth:** `verifyParticipantJWT`

**Path Parameters:**
- `:as` — Role: `"provider"` or `"consumer"`

**Flow:**
1. Get authenticated participant
2. Fetch bilateral and/or ecosystem contracts based on role:
   - `GET {CONTRACT_SERVICE_URL}/contracts/bilateral?participantId={id}`
   - `GET {CONTRACT_SERVICE_URL}/contracts/ecosystem?participantId={id}`
3. Return available exchanges

**Response:** `200 OK` with list of available exchanges

### Consent Receipt Format (ISO/IEC 29184)

**Method:** `Consent.toReceipt()`

Transform a consent document into an ISO/IEC 29184-compliant receipt:

```json
{
  "version": "KI-CR-v1.1.0",
  "jurisdiction": "...",
  "consentTimestamp": "ISO datetime",
  "collectionMethod": "...",
  "consentReceiptID": "consent._id",
  "publicKey": "...",
  "language": "en",
  "piiPrincipalId": "user email",
  "piiControllers": [...],
  "policyUrl": "...",
  "services": [...],
  "sensitive": false,
  "spiCat": []
}
```

Use actual schema fields — not `this.json` (fixing original bug).

### Parent/Child Inclusion

When `includeChildren=true`:
- For each consent, populate the `child` field
- If the child also has a `child`, recursively populate (up to a configurable depth limit to prevent infinite loops)

### Files to Create/Modify

```
src/controllers/consentsController.ts  # getUserConsents, getUserConsentById,
                                        # getParticipantConsents, getAvailableExchanges
src/routes/consent.ts                  # Querying routes
```

## Acceptance Criteria

1. [ ] `GET /consents/me` returns paginated consents for the authenticated user
2. [ ] `GET /consents/me` `totalCount` is filtered to the user's consents (not global)
3. [ ] `GET /consents/me?receipt=true` returns consents in ISO/IEC 29184 receipt format
4. [ ] `GET /consents/me?includeChildren=true` includes child consent chain
5. [ ] `GET /consents/me/:id` returns a single consent owned by the user
6. [ ] `GET /consents/participants/:userId/` returns consents scoped to the participant
7. [ ] `GET /consents/participants/:userId/:id` returns a specific consent for the participant
8. [ ] `GET /consents/exchanges/user` discovers available exchanges for the user
9. [ ] `GET /consents/exchanges/:as` discovers available exchanges for the participant
10. [ ] Pagination works correctly with `page` and `limit` parameters
11. [ ] Integration tests cover all querying endpoints with various filters

## Known Issues to Address

1. **Global count bug** (original bug 18.3): Use the user-filtered query for `totalCount`, not `Consent.countDocuments()` with no filter.
2. **`toReceipt()` references non-existent field** (original bug 18.5): Implement the receipt transformation using actual schema fields.
3. **Parent/child recursive depth**: Add a configurable maximum depth to prevent infinite recursion if circular references exist.
