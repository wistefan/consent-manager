# TICKET-009: Consent Lifecycle Operations

## Metadata

| Field | Value |
|-------|-------|
| **Ticket ID** | TICKET-009 |
| **Title** | Consent Lifecycle Operations |
| **Priority** | High |
| **Complexity** | M (Medium) |
| **Dependencies** | TICKET-008 |
| **Related User Stories** | US-CM-006, US-CM-007, US-CM-008, US-CM-009, US-CM-010, US-CM-024 |

## Description

Implement the operations that move a consent through its lifecycle after creation: revocation,
refusal, termination, re-confirmation and resume. Every one of them goes through the state machine
from TICKET-008 and appends an event.

Complexity drops from L to M: the data-exchange triggering that these operations carried is gone,
and with it the ordering hazard it created.

## Technical Details

### Endpoints

#### 1. Revoke Consent — `POST /consents/{id}/revoke` (US-CM-006)

**Auth:** `USER`

**Flow:**
1. Load the consent; 404 if unknown or not owned by the caller.
2. Guard the transition `GRANTED → REVOKED`; 400 with both states named if invalid.
3. Set `status = REVOKED`, `consented = false` via the state factory.
4. Append a `CONSENT_REVOKED` event with the user as `actor`.
5. Emit a state-change notification (TICKET-020).

**Response:** `200 OK` with the updated consent.

Revocation is effective immediately in this service's records. **Whether it is effective in
practice depends on participants being told and acting on it** — hence step 5, and the explicit
non-goal recorded in the architecture documentation. `DELETE` was the pre-revision verb; `POST
.../revoke` is used instead because nothing is deleted and the operation is not idempotent in the
way `DELETE` implies.

#### 2. Refuse Consent — `POST /consents/{id}/refuse` (US-CM-007)

**Auth:** `USER`

Guards `PENDING → REFUSED` or `DRAFT → REFUSED`. Sets `consented = false`, appends
`CONSENT_REFUSED`. Refusing an already `GRANTED`, `REVOKED` or `REFUSED` consent is a 400.

This is how a user declines a participant's consent request (TICKET-008, US-CM-004).

#### 3. Terminate Consent — `POST /consents/{id}/terminate` (US-CM-008)

**Auth:** `PARTICIPANT`

**Flow:**
1. Load the consent; 404 if unknown.
2. Verify the caller is the provider or consumer on that consent; otherwise **403**.
3. Guard `GRANTED → TERMINATED`.
4. Append `CONSENT_TERMINATED` with the participant as `actor`.
5. Notify the user (TICKET-013) — a consent ended by the organization is something the data subject
   should learn about.

Termination is the organizational counterpart to revocation: the arrangement ends for a reason on
the participant's side, such as contract expiry. It does not represent the user withdrawing.

#### 4. Re-confirm Consent — `POST /consents/{id}/re-confirm` (US-CM-009)

**Auth:** `USER`

**Flow:**
1. Load the consent; 404 if unknown or not owned by the caller.
2. Require `status = GRANTED`; otherwise 400.
3. Append a `CONSENT_RE_CONFIRMED` event. **The status does not change** — re-confirmation is an
   affirmation recorded in the audit trail, not a transition.

The pre-revision version took a `triggerDataExchange` query parameter; it is removed, which leaves
this operation as a pure event append.

#### 5. Resume Consent — `POST /consents/{id}/resume` (US-CM-010)

**Auth:** `USER` or `CATALOG`

**Request Body:** empty for a `USER` caller. For a `CATALOG` caller:
```json
{ "userIdentifier": "string (required — global identifier)" }
```

**Flow:**
1. Load the consent; 404 if unknown.
2. For a `USER` caller, require ownership. For a `CATALOG` caller, require that `userIdentifier`
   matches the consent's user.
3. Guard `PENDING → GRANTED` or `DRAFT → GRANTED`; a consent already `GRANTED`, `REVOKED`,
   `REFUSED` or `TERMINATED` is a 400.
4. Apply the same re-grant handling as TICKET-008 step 5: revoke and link any existing `GRANTED`
   consent for the same (user, privacy notice) in the same transaction.
5. Set `GRANTED` / `consented = true`, append `CONSENT_RESUMED`.
6. Sign the receipt (TICKET-011) and emit a state-change notification (TICKET-020).

This is how a user accepts a participant's consent request. The whole operation runs in **one
transaction** — the pre-revision draft path persisted the status only after triggering a downstream
action, so the database did not reflect the granted state when that action ran (original bug 18.8).
With the downstream call gone and the write transactional, the hazard is designed out.

The pre-revision body also carried `email`, `userIdentifier` and
`participantSelfDescriptionURL` in order to create a `UserIdentifier` on the fly. There is nothing
to create: the user already exists and the participant link is managed by TICKET-004.

### Shared Concerns

**Ownership and authority are checked on every operation.** The original did not consistently do
this. `USER` operations require the caller to be the consent's data subject; `PARTICIPANT`
operations require the caller to be provider or consumer on that consent. Role alone is never
sufficient.

**Every operation is transactional**, appending its event and writing the status in the same unit of
work. The two cannot diverge, which removes the "missing `await` on save" class of defect
(original bug 18.4) structurally rather than by review discipline.

**Events are append-only** and carry `actor` — the identifier of whoever caused the transition. The
pre-revision event record had no actor field, so the audit trail could not answer who terminated a
consent.

### Files to Create

```
src/main/java/eu/prometheusx/consentmanager/
  api/ConsentLifecycleController.java
  service/ConsentLifecycleService.java
```

Both build on `ConsentStateMachine` and `ConsentFactory` from TICKET-008.

## Acceptance Criteria

1. [ ] `POST /consents/{id}/revoke` moves `GRANTED → REVOKED` and sets `consented = false`
2. [ ] `POST /consents/{id}/refuse` moves `PENDING`/`DRAFT → REFUSED`
3. [ ] `POST /consents/{id}/terminate` moves `GRANTED → TERMINATED` for a party to the consent
4. [ ] `POST /consents/{id}/re-confirm` appends an event and leaves the status unchanged
5. [ ] `POST /consents/{id}/resume` moves `PENDING`/`DRAFT → GRANTED`
6. [ ] Resume revokes and links any existing `GRANTED` consent in the same transaction
7. [ ] Every invalid transition returns 400 naming the current and attempted states
8. [ ] A user operating on another user's consent receives 404
9. [ ] A participant terminating a consent it is not party to receives 403
10. [ ] A `CATALOG` caller must supply a `userIdentifier` matching the consent's user
11. [ ] Each operation appends exactly one event carrying the acting identity
12. [ ] Status and event are written in one transaction; neither can persist without the other
13. [ ] Termination notifies the data subject
14. [ ] Revocation emits a participant notification
15. [ ] Parameterized tests cover every (operation × starting state) pair, valid and invalid
16. [ ] A concurrent revoke and re-grant leaves exactly one `GRANTED` consent

## Changes from the Pre-Revision Ticket

- `DELETE /consents/{id}` becomes `POST /consents/{id}/revoke`; nothing is deleted.
- `triggerDataExchange` is removed from re-confirm and resume.
- Resume no longer creates a `UserIdentifier` from request-body data.
- `CATALOG` authority replaces the `x-catalog-consent-key` shared secret on resume.
- Events gain an `actor` field.
- Original bugs 18.4 (missing `await`) and 18.8 (resume not persisting before the downstream call)
  are resolved by transactional boundaries and the removal of the downstream call.
