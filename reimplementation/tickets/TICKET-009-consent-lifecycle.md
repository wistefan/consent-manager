# TICKET-009: Consent Lifecycle Operations

## Metadata

| Field | Value |
|-------|-------|
| **Ticket ID** | TICKET-009 |
| **Title** | Consent Lifecycle Operations |
| **Priority** | High |
| **Complexity** | L (Large) |
| **Dependencies** | TICKET-008 |
| **Related User Stories** | US-CM-006, US-CM-007, US-CM-008, US-CM-009, US-CM-010, US-CM-024 |

## Description

Implement the consent lifecycle operations that manage consent state after it has been granted: revocation, refusal, termination, re-confirmation, and resume. Each operation transitions the consent through the state machine defined in TICKET-008.

## Technical Details

### Endpoints

#### 1. Revoke Consent — `DELETE /consents/:id`

**Auth:** `verifyUserJWT`

**Flow:**
1. Find consent by ID
2. Verify the authenticated user owns this consent
3. Validate state transition: must be in `granted` state
4. Update consent:
   - `status = "revoked"`
   - `consented = false`
   - Add event: `{ eventType: "explicit", eventState: "consent revoked", eventTime: new Date().toISOString() }`
5. `await consent.save()`
6. Return updated consent

**Response:** `200 OK` with updated consent

#### 2. Refuse Consent — `POST /consents/:consentId/refuse`

**Auth:** `verifyUserJWT`

**Flow:**
1. Find consent by ID
2. Verify the authenticated user owns this consent
3. Validate state transition: must be in `pending` or `draft` state
4. Update consent:
   - `status = "refused"`
   - `consented = false`
   - Add refusal event
5. `await consent.save()` (fixing original missing-await bug)
6. Return updated consent

**Response:** `200 OK` with updated consent

#### 3. Terminate Consent — `POST /consents/:consentId/terminate`

**Auth:** `verifyParticipantJWT`

**Flow:**
1. Find consent by ID
2. Verify the authenticated participant is either the data provider or consumer
3. Validate state transition: must be in `granted` state
4. Update consent:
   - `status = "terminated"`
   - `consented = false`
   - Add termination event
5. `await consent.save()` (fixing original missing-await bug)
6. Return updated consent

**Response:** `200 OK` with updated consent

#### 4. Re-confirm Consent — `POST /consents/:consentId/re-confirm`

**Auth:** `verifyUserJWT`

**Flow:**
1. Find consent by ID
2. Verify the authenticated user owns this consent
3. Validate: consent must be in `granted` state (re-confirmation is an affirmation, not a state change)
4. Add re-confirmation event:
   ```json
   {
     "eventType": "explicit",
     "eventState": "consent re-confirmed",
     "eventTime": "ISO timestamp"
   }
   ```
5. Save consent
6. Optionally trigger data exchange if `triggerDataExchange=true` query param is set
7. Return updated consent

**Response:** `200 OK` with updated consent

#### 5. Resume Consent — `POST /consents/:consentId/resume`

**Auth:** `verifyUserJWT` or `verifyParticipantJWT`

**Request Body:**
```json
{
  "email": "string (optional)",
  "userIdentifier": "string (optional)",
  "participantSelfDescriptionURL": "string (optional)"
}
```

**Flow:**
1. Find consent by ID
2. Validate state: must be in `pending` or `draft` state
3. **For `pending` state:**
   - Transition to `granted`
   - Set `consented = true`
   - Add grant event
   - Create/link UserIdentifier from provided data if needed
   - Save consent
   - Optionally trigger data exchange
4. **For `draft` state:**
   - Transition to `granted`
   - Set `consented = true`
   - Add grant event
   - **Save consent before triggering data exchange** (fixing original bug)
   - Optionally trigger data exchange
5. Return updated consent

**Response:** `200 OK` with updated consent

### State Machine Integration

All lifecycle operations must use the state machine utility from TICKET-008:

```typescript
import { validateTransition } from '../utils/consentStateMachine';

// Before any state change:
validateTransition(consent.status, newStatus);
// Throws BadRequestError if transition is invalid
```

### Event Logging

Every lifecycle operation appends an event to the consent's `event` array:

```typescript
interface ConsentEvent {
  eventTime: string;       // ISO 8601 datetime
  validityDuration: string;
  eventType: string;       // "explicit"
  eventState: string;      // Describes what happened
}
```

Standard event states:
- `"consent given"` — Consent granted
- `"consent revoked"` — User revoked consent
- `"consent refused"` — User refused consent
- `"consent terminated"` — Participant terminated consent
- `"consent re-confirmed"` — User re-confirmed consent
- `"consent resumed"` — Consent resumed from pending/draft

### Files to Create/Modify

```
src/controllers/consentsController.ts  # revokeConsent, refuseConsent, terminateConsent,
                                        # reConfirmConsent, resumeConsent
src/routes/consent.ts                  # Lifecycle routes
```

## Acceptance Criteria

1. [ ] `DELETE /consents/:id` revokes a granted consent (sets status to `revoked`, consented to `false`)
2. [ ] `POST /consents/:consentId/refuse` refuses a pending/draft consent
3. [ ] `POST /consents/:consentId/terminate` terminates a granted consent (participant-only)
4. [ ] `POST /consents/:consentId/re-confirm` adds a re-confirmation event to a granted consent
5. [ ] `POST /consents/:consentId/resume` transitions pending/draft consent to granted
6. [ ] All operations enforce state machine transitions (invalid transitions return 400)
7. [ ] All operations append an event to the consent's event array
8. [ ] Revocation requires the owning user's authentication
9. [ ] Termination requires an authorized participant (provider or consumer)
10. [ ] Resume from `draft` saves the consent before triggering data exchange
11. [ ] All `consent.save()` calls are properly `await`ed
12. [ ] Integration tests cover each lifecycle operation with valid and invalid state transitions

## Known Issues to Address

1. **Missing `await` on `consent.save()`** (original bugs 18.4): Both `refuseConsent` and `terminateConsent` must `await` the save operation.
2. **`resumeConsent` draft case doesn't save before data exchange** (original bug 18.8): In the `draft` case, call `await consent.save()` before triggering `dataExchanges(consentId)` so the database reflects the `"granted"` status.
3. **No ownership verification**: The original code does not always verify that the requesting user/participant owns the consent. Add ownership checks to all operations.
