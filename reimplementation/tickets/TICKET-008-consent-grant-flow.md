# TICKET-008: Consent Grant Flow

## Metadata

| Field | Value |
|-------|-------|
| **Ticket ID** | TICKET-008 |
| **Title** | Consent Grant Flow |
| **Priority** | Critical |
| **Complexity** | L (Large) |
| **Dependencies** | TICKET-002, TICKET-003, TICKET-007, TICKET-011 |
| **Related User Stories** | US-CM-003, US-CM-004, US-CM-021, US-CM-024 |

## Description

Implement consent creation: the data subject granting or refusing consent against a privacy notice,
and a participant *requesting* consent from a user.

This was the most complex ticket in the pre-revision plan (XL). It is now **L**. Three things left
with the redesign:

- **Two authentication flows became one.** With external identity there is exactly one way to
  identify the data subject: their token.
- **The four email-reconciliation cases are gone.** They existed to decide which of several
  `UserIdentifier` rows belonged to which `User`. Users now have one global identifier.
- **Inline data exchange is gone.** `triggerDataExchange` no longer exists in any form.

## Technical Details

### The Core Principle

**Only the data subject grants consent.** The pre-revision `POST /consents` flow authenticated with
an `x-user-key` header supplied by the calling participant, which meant a participant could create a
granted consent on a user's behalf. That is removed. A participant may now *ask*; only the user may
*answer*.

### Endpoint 1: Grant or Refuse Consent — `POST /consents` (US-CM-003)

**Auth:** `USER`

**Request Body:**
```json
{
  "privacyNoticeId": "uuid (required)",
  "decision": "GRANTED | REFUSED (required)",
  "purposes": ["optional subset of the notice's purposes"],
  "data": ["optional subset of the notice's data categories"]
}
```

**Flow:**
1. Resolve the user from the token. The body carries **no** user identifier, email or user key.
2. Load the privacy notice; 404 if unknown, **400** if archived — consent cannot be granted against
   terms that are no longer in force.
3. Resolve provider and consumer from the notice.
4. If `purposes` or `data` are supplied, verify each is a subset of what the notice offers;
   otherwise **400**. A consent must never claim a scope wider than the notice.
5. **Re-grant handling** (US-CM-021): if a `GRANTED` consent already exists for this
   (user, privacy notice):
   - transition it to `REVOKED` with a `CONSENT_REVOKED` event;
   - set `parentConsentId` on the new consent to that consent's ID.
   Both steps run in **one transaction** with the new consent's creation, so the partial unique
   index on `(user_id, privacy_notice_id) WHERE status = 'GRANTED'` is never violated and no window
   exists in which the user has two active consents or none.
6. Freeze the notice terms into the consent `snapshot`. The consent must stay auditable after the
   notice is archived.
7. Create the consent with status and `consented` set together by a single factory method:
   `GRANTED`/`true` or `REFUSED`/`false`.
8. Append the corresponding event (`CONSENT_GIVEN` or `CONSENT_REFUSED`).
9. Sign the consent receipt (TICKET-011).
10. Emit a state-change notification (TICKET-020).

**Response:** `201 Created` with the consent representation.

### Endpoint 2: Request Consent from a User — `POST /consents/requests` (US-CM-004)

**Auth:** `PARTICIPANT`

**Request Body:**
```json
{
  "privacyNoticeId": "uuid (required)",
  "userIdentifier": "string (required — global identifier)"
}
```

**Flow:**
1. Verify the caller is the provider or consumer named on the notice; otherwise **403**.
2. Verify the caller is linked to that user; otherwise **404** (not 403 — a 403 would confirm the
   identifier exists).
3. Reject with **409** if a `GRANTED` or `PENDING` consent already exists for this
   (user, privacy notice).
4. Create the consent with status `PENDING` and `consented = false`.
5. Append a `CONSENT_REQUESTED` event recording the requesting participant as `actor`.
6. Notify the user (TICKET-013) that a consent request is waiting.

**Response:** `201 Created` with the pending consent.

A `PENDING` consent confers **nothing**. It is a request, not a consent, and `consented` is `false`
until the user acts on it through `POST /consents/{id}/resume` or the refuse operation (TICKET-009).

This is the deliberate replacement for the pre-revision participant-initiated grant: the use case
survives, the ability for a participant to self-grant does not.

### Consent State Machine (US-CM-024)

A single table-driven guard, used by this ticket and TICKET-009. No transition anywhere in the
codebase bypasses it.

| From | To | Trigger |
|------|----|---------|
| — | `GRANTED` | User grants |
| — | `REFUSED` | User refuses |
| — | `PENDING` | Participant requests consent |
| — | `DRAFT` | Consent prepared, awaiting user action |
| `PENDING` | `GRANTED`, `REFUSED` | User acts on the request |
| `DRAFT` | `GRANTED`, `REFUSED` | User completes the draft |
| `GRANTED` | `REVOKED` | User revokes |
| `GRANTED` | `TERMINATED` | Provider or consumer terminates |
| `GRANTED` | `EXPIRED` | System, on TTL elapse |

Every other transition is invalid and raises `BadRequestException` naming the current state and the
attempted target. `REVOKED`, `REFUSED`, `TERMINATED` and `EXPIRED` are terminal: a new decision
creates a new consent with a parent link, never a resurrection of the old one.

`consented` is not an independently settable field. A single factory derives it from the target
status, so the "refused but recorded as granted" defect (original bug 18.1) has no way to occur.

### Consent Lineage (US-CM-021)

Lineage is a **single upward link**: each consent may name the consent it supersedes via
`parentConsentId`. There is no reciprocal `child` column to keep in sync, which is what the original
got wrong — the schema declared `child` while the code pushed to `children` (original bug 18.2).

The chain is walked from the newest consent backwards. TICKET-010 exposes it with a bounded depth.

### Files to Create

```
src/main/java/eu/prometheusx/consentmanager/
  api/ConsentController.java              # grant, refuse
  api/ConsentRequestController.java       # participant consent requests
  service/ConsentService.java
  service/ConsentStateMachine.java        # the single transition guard
  service/ConsentFactory.java             # status + consented set together
```

**Authored in the API specification, not hand-written** (TICKET-015): the request and response
models below are generated from these schemas. Controllers implement the generated interfaces.

```
api/components/schemas/GrantConsent.yaml
api/components/schemas/ConsentRequest.yaml
api/components/schemas/Consent.yaml
api/paths/consents.yaml
api/paths/consent-requests.yaml
```

## Acceptance Criteria

1. [ ] `POST /consents` identifies the data subject solely from the token
2. [ ] A user identifier, email or user key in the request body is ignored, not honoured
3. [ ] There is no endpoint by which a participant can create a `GRANTED` consent
4. [ ] `decision: REFUSED` produces `status = REFUSED` and `consented = false`
5. [ ] Granting against an archived privacy notice is rejected with 400
6. [ ] Purposes or data outside the notice's scope are rejected with 400
7. [ ] Re-granting revokes the prior consent and links the new one via `parentConsentId`
8. [ ] Revocation and creation happen in one transaction; no state has two `GRANTED` consents
9. [ ] The consent snapshot preserves the notice terms after the notice is archived
10. [ ] `POST /consents/requests` creates a `PENDING` consent with `consented = false`
11. [ ] A participant that is not a party to the notice receives 403
12. [ ] A participant not linked to the user receives 404
13. [ ] A duplicate request for an existing `GRANTED` or `PENDING` consent returns 409
14. [ ] The requesting participant is recorded as `actor` on the `CONSENT_REQUESTED` event
15. [ ] The state machine rejects every invalid transition with a message naming both states
16. [ ] `consented` cannot be set independently of `status` anywhere in the codebase
17. [ ] Every creation appends exactly one event
18. [ ] Parameterized tests cover the full transition matrix, valid and invalid
19. [ ] Integration tests use real tokens from the Testcontainers IDP

## Removed Scope

| Removed | Reason |
|---------|--------|
| `POST /consents` via `x-user-key` header | A participant may no longer act as the data subject |
| `POST /consents/user` as a second grant path | Merged into the single grant endpoint |
| The four email-reconciliation cases (2.a–2.d) | No `UserIdentifier` entity to reconcile |
| `GET /consents/emailverification` | Email verification is the IDP's responsibility (original bug 18.6 removed with it) |
| `triggerDataExchange` query parameter | Data exchange is out of scope |
| `emailReattached` fall-through handling (original bug 18.9) | The code path no longer exists |
