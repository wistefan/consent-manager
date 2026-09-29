# TICKET-020: Participant Consent Notifications

## Metadata

| Field | Value |
|-------|-------|
| **Ticket ID** | TICKET-020 |
| **Title** | Participant Consent Notifications |
| **Priority** | High |
| **Complexity** | M (Medium) |
| **Dependencies** | TICKET-005, TICKET-008, TICKET-009 |
| **Related User Stories** | US-CM-025 |

## Description

Deliver consent state changes to the participants bound by them, over a transactional outbox with
webhook delivery.

## Why This Exists

Removing data exchange from scope removed the Consent Manager from the data path. It used to hold
the encrypted consent and forward it to the provider's and consumer's connectors, which meant it
could simply decline to forward once a consent was withdrawn. It now only records.

That leaves a gap: **a revocation nobody is told about is not an effective revocation.** A user
withdraws consent, the record changes, and the provider keeps processing because nothing informed
it. The user has exercised a right that produced no effect.

Participants are expected to consult `GET /consents/check` (TICKET-010) before processing, and data
space governance must require it. But polling alone leaves a window between withdrawal and the next
check, and that window is exactly when processing is no longer lawful. Push closes it.

This is the one capability worth carrying over from the removed DSC integration, and it is a consent
concern rather than a data-exchange one: it communicates a change in permission, never data.

## Technical Details

### Transactional Outbox

Notifications are written to an outbox table **in the same transaction** as the consent state
change. A consent cannot change state without its notification being durably queued, and a
notification cannot exist for a state change that was rolled back.

Sending inline instead would mean either holding a database transaction open across an HTTP call to
an unreliable third party, or losing notifications whenever the process dies between commit and
send. Neither is acceptable for a record of permission.

```sql
CREATE TABLE notification_outbox (
    id                uuid PRIMARY KEY,
    consent_id        uuid NOT NULL REFERENCES consents (id) ON DELETE CASCADE,
    participant_id    uuid NOT NULL REFERENCES participants (id) ON DELETE CASCADE,
    event_type        text NOT NULL,
    payload           jsonb NOT NULL,
    status            text NOT NULL DEFAULT 'PENDING',
    attempts          integer NOT NULL DEFAULT 0,
    next_attempt_at   timestamptz NOT NULL DEFAULT now(),
    last_error        text,
    created_at        timestamptz NOT NULL DEFAULT now(),
    delivered_at      timestamptz,
    CONSTRAINT notification_outbox_status_check
        CHECK (status IN ('PENDING','DELIVERED','FAILED','ABANDONED'))
);
CREATE INDEX notification_outbox_due_idx
    ON notification_outbox (next_attempt_at)
    WHERE status = 'PENDING';
```

Delivered as migration `V2__notification_outbox.sql`.

### Notified Events

| Event | Recipients | Why |
|-------|-----------|-----|
| `CONSENT_GRANTED` | Provider and consumer | Permission now exists |
| `CONSENT_REVOKED` | Provider and consumer | **Permission withdrawn — the critical one** |
| `CONSENT_TERMINATED` | The other party | The arrangement was ended by one side |
| `CONSENT_EXPIRED` | Provider and consumer | Permission lapsed |

`PENDING`, `DRAFT`, `REFUSED` and re-confirmation are not notified: no permission changed, and the
requesting participant already knows it asked.

### Payload

```json
{
  "eventType": "CONSENT_REVOKED",
  "eventId": "01J8X...",
  "occurredAt": "2026-09-29T14:02:11Z",
  "consentId": "3f9a...",
  "privacyNoticeId": "b21c...",
  "userIdentifier": "did:example:123",
  "status": "REVOKED",
  "consented": false
}
```

The payload carries **no personal data beyond the user identifier the recipient already holds**, and
no consent content. It says what changed and where to look; the recipient calls
`GET /consents/{id}` or `GET /consents/check` for detail, under its own authentication.

### Delivery

`POST` to the recipient's `endpoints.consentNotification` (TICKET-005). Participants without one
configured are skipped, and the outbox row is marked `ABANDONED` with a reason rather than retried
forever.

**Authenticity:** each request carries a detached JWS over the payload in an
`X-Consent-Signature` header, using the signing keys from TICKET-011 and verifiable against
`GET /.well-known/jwks.json`. A webhook that cannot be authenticated is a way to trick a participant
into believing consent was withdrawn — or that it was granted.

**Replay protection:** `eventId` is stable per event and repeated across retries. Recipients are
documented as needing to treat delivery as at-least-once and to deduplicate on `eventId`.

**Retry:** exponential backoff with jitter — 1m, 5m, 15m, 1h, 6h, 24h — then `ABANDONED`.
2xx is success; 4xx other than 408/429 is a permanent failure and is abandoned immediately, since
retrying a rejected payload cannot succeed.

**Timeouts:** short connect and read timeouts. A slow recipient must not occupy a worker; the
scheduler moves on and retries.

**Isolation:** one participant's unreachable endpoint must never delay delivery to another. Poll due
rows in batches and dispatch concurrently with a bounded pool, taking rows with
`FOR UPDATE SKIP LOCKED` so multiple replicas can drain the outbox without duplicating work or
blocking one another.

### Operability

- Metrics: queue depth, delivery latency, attempts, failures and abandonments per participant.
- `GET /participants/me/notifications` (auth `PARTICIPANT`) lists the caller's recent delivery
  attempts and outcomes, so integrators can debug their own endpoint without operator help.
- Abandoned notifications are logged at WARN with the participant, the consent and the reason.

### Files to Create

```
src/main/resources/db/migration/V2__notification_outbox.sql
src/main/java/eu/prometheusx/consentmanager/
  notification/ParticipantNotificationService.java   # enqueue, in-transaction
  notification/NotificationOutboxRepository.java
  notification/NotificationDispatcher.java           # scheduled drain, backoff
  notification/NotificationPayload.java
  notification/NotificationConfiguration.java
  api/NotificationStatusController.java
```

## Acceptance Criteria

1. [ ] The outbox row is written in the same transaction as the consent state change
2. [ ] A rolled-back state change leaves no outbox row
3. [ ] A committed state change always leaves an outbox row, even if the process dies before sending
4. [ ] Grant, revoke, terminate and expire produce notifications; pending, draft, refuse and re-confirm do not
5. [ ] Termination notifies the other party
6. [ ] The payload carries no consent content beyond identifiers and status
7. [ ] Each delivery carries a JWS signature verifiable against the published JWKS
8. [ ] `eventId` is stable across retries and documented as an at-least-once deduplication key
9. [ ] Retries follow the documented backoff and stop at `ABANDONED`
10. [ ] A 4xx other than 408/429 is abandoned immediately rather than retried
11. [ ] A participant with no notification endpoint is abandoned with a reason, not retried
12. [ ] One unreachable endpoint does not delay delivery to any other participant
13. [ ] Multiple replicas drain the outbox without duplicate delivery, via `SKIP LOCKED`
14. [ ] Delivery metrics are exposed per participant
15. [ ] `GET /participants/me/notifications` returns only the caller's own delivery history
16. [ ] Integration tests cover success, 4xx, 5xx, timeout, and abandonment against a WireMock recipient
17. [ ] A concurrency test asserts two replicas deliver each notification exactly once

## Scope Boundary

This ticket delivers **notifications about consent**, never data and never instructions. It does not
reintroduce data exchange, it does not call a participant's connector, and it does not act on a
participant's behalf. If a recipient chooses to stop processing on receiving a revocation, that is
the recipient's action within its own systems.

The Consent Manager remains a recorder. This makes its records reach the parties bound by them.
