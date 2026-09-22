# TICKET-008: Consent Grant Flow

## Metadata

| Field | Value |
|-------|-------|
| **Ticket ID** | TICKET-008 |
| **Title** | Consent Grant Flow |
| **Priority** | Critical |
| **Complexity** | XL (Extra Large) |
| **Dependencies** | TICKET-002, TICKET-003, TICKET-006, TICKET-007, TICKET-011, TICKET-013 |
| **Related User Stories** | US-CM-003, US-CM-004, US-CM-005, US-CM-020, US-CM-021, US-CM-023, US-CM-024 |

## Description

Implement the core consent grant logic — the most complex feature in the Consent Manager. This covers two authentication flows (User Key and User JWT), email-based verification, user interaction scenarios (4 reconciliation cases), parent/child consent inheritance for re-grants, optional inline data exchange triggering, and the consent state machine foundation.

## Technical Details

### Endpoint 1: Grant Consent via User Key — `POST /consents`

**Auth:** `verifyUserKey`

**Query Parameters:**
- `triggerDataExchange` (boolean, optional) — If true, trigger inline data exchange after granting

**Request Body:**
```json
{
  "privacyNoticeId": "string (required)",
  "userId": "string (required — user ID)",
  "email": "string (required — user's email)",
  "dataProvider": "string (required — base64-encoded provider self-description URL)",
  "dataConsumer": "string (required — base64-encoded consumer self-description URL)",
  "data": ["array of data items (optional)"],
  "purposes": ["array of purposes (optional)"],
  "event": "string (optional — 'given' or 'refused')"
}
```

**Flow:**
1. Validate the privacy notice exists and is not archived
2. Decode provider and consumer self-description URLs (base64)
3. Resolve provider and consumer Participant documents
4. Resolve provider-side UserIdentifier for the user at the provider
5. Resolve consumer-side UserIdentifier for the user at the consumer
6. **User Interaction Scenarios** (see below)
7. Create the Consent document with:
   - `status: "granted"` (or `"refused"` based on event)
   - `consented: true` (or `false` for refused)
   - `event`: lifecycle event entry
   - All data from privacy notice and request body
8. **Parent/Child Inheritance:** If the user already has a consent for the same privacy notice:
   - Set the new consent's `parent` to the existing consent's ID
   - Set the existing consent's `child` to the new consent's ID (singular field)
   - The existing consent becomes inactive
9. **Inline Data Exchange:** If `triggerDataExchange=true`, call `triggerDataExchangeByConsentId()`
10. Return the created consent

### Endpoint 2: Grant Consent via User JWT — `POST /consents/user`

**Auth:** `verifyUserJWT`

**Flow:** Same as above, but:
- User is identified from the JWT token (`req.user`)
- No need for `userId` or `email` in the request body
- UserIdentifiers are resolved for the authenticated user

### User Interaction Scenarios (Email Verification Cases)

When a consent is granted, the system must reconcile the user's identity across the provider and consumer. Four scenarios exist:

**Case 2.a — Both UserIdentifiers are linked to the same User:**
- Straightforward: grant consent directly
- No email verification needed

**Case 2.b — Provider UserIdentifier is linked to a User, consumer is not:**
- Link consumer UserIdentifier to the provider's User
- Grant consent directly

**Case 2.c — Both UserIdentifiers exist but are linked to different Users (or one is unlinked):**
- **Sub-case 2.c1:** Consumer user has no password → merge into provider user
- **Sub-case 2.c2:** Consumer user has a password → send email verification to confirm identity
- Create consent in `"pending"` state
- Wait for email verification callback

**Case 2.d — Neither UserIdentifier is linked to a User:**
- Send email verification to create and link the user
- Create consent in `"pending"` state

### Email Verification Endpoint — `GET /consents/emailverification`

**Auth:** None (public — callback from email link)

**Query Parameters:**
```
token=<verification_token>
consentId=<consent_id>
userId=<user_id>
... (additional context parameters)
```

**Flow:**
1. Verify the token
2. Find the consent by ID
3. Execute the appropriate reconciliation case based on parameters
4. Update consent status from `"pending"` to `"granted"`
5. Set `consented: true`
6. Redirect user to a success page or PDI

**Security Note:** The original endpoint has no authentication (bug 18.6). In the reimplementation, use a signed, time-limited token in the email link to prevent forgery.

### Consent State Machine

Define valid state transitions:

| From State | To State | Trigger |
|------------|----------|---------|
| (none) | `pending` | Consent created, awaiting email verification |
| (none) | `draft` | Consent created, awaiting user action |
| (none) | `granted` | Consent directly granted |
| (none) | `refused` | Consent directly refused |
| `pending` | `granted` | Email verification completed |
| `pending` | `refused` | User refuses via email |
| `draft` | `granted` | User completes consent via resume |
| `draft` | `refused` | User refuses consent |
| `granted` | `revoked` | User revokes consent |
| `granted` | `terminated` | Participant terminates consent |
| `granted` | `expired` | Consent reaches expiry |

Enforce these transitions in a state machine utility. Invalid transitions should throw a `BadRequestError`.

### Parent/Child Consent Linking

When a user re-grants consent for the same privacy notice:
1. Find the existing consent for the same user + privacyNotice + provider + consumer
2. Set `existingConsent.child = newConsent._id` (singular ObjectId)
3. Set `newConsent.parent = existingConsent._id`
4. The existing consent's `consented` flag remains as-is (it may have been revoked)

### Inline Data Exchange

When `triggerDataExchange=true`:
1. After consent is granted, call `triggerDataExchangeByConsentId(consentId)`
2. This function is implemented in TICKET-012 — for now, define the interface and make it a pluggable dependency

### Files to Create/Modify

```
src/controllers/consentsController.ts  # giveConsent, giveConsentUser, emailReattached
src/routes/consent.ts                  # Consent grant routes
src/utils/consentStateMachine.ts       # State transition enforcement
```

## Acceptance Criteria

1. [ ] `POST /consents` creates a consent via User Key authentication
2. [ ] `POST /consents/user` creates a consent via User JWT authentication
3. [ ] Both flows correctly resolve provider and consumer UserIdentifiers
4. [ ] User interaction case 2.a grants consent directly
5. [ ] User interaction case 2.b links consumer UserIdentifier and grants
6. [ ] User interaction case 2.c sends email verification for identity conflicts
7. [ ] User interaction case 2.d sends email verification for unknown users
8. [ ] `GET /consents/emailverification` processes the verification callback
9. [ ] Email verification uses signed, time-limited tokens
10. [ ] Parent/child linking works correctly (singular `child` field)
11. [ ] State machine enforces valid transitions only
12. [ ] `triggerDataExchange=true` triggers inline data exchange after grant
13. [ ] Refused event creates a consent with `status: "refused"` and `consented: false`
14. [ ] Integration tests cover all 4 user interaction scenarios

## Known Issues to Address

1. **`giveConsentUser` refused-but-granted bug** (original bug 18.1): When `event="refused"`, the consent must be created with `status: "refused"` and `consented: false`, not `"granted"` / `true`.
2. **Parent update uses wrong field** (original bug 18.2): Use singular `child` field (ObjectId), not `children` (array push).
3. **Email verification has no authentication** (original bug 18.6): Implement signed, time-limited verification tokens.
4. **`emailReattached` falls through** (original bug 18.9): Handle the case where both users have passwords (case 2.c2) properly — do not fall through to `undefined`.
