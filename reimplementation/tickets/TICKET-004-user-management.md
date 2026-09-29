# TICKET-004: User Management

## Metadata

| Field | Value |
|-------|-------|
| **Ticket ID** | TICKET-004 |
| **Title** | User Management |
| **Priority** | Critical |
| **Complexity** | M (Medium) |
| **Dependencies** | TICKET-002, TICKET-003 |
| **Related User Stories** | US-UM-001, US-UM-004, US-UM-005, US-UM-006, US-UM-007, US-UM-008, US-UM-011, US-UM-012, US-UM-014 |

## Description

Implement user records and the user↔participant link. A user exists only to fulfil the role of a
data subject: there is no account, no password, and no credential of any kind. Identity is external
(TICKET-003); this ticket owns the record that consents hang off and the participant links that say
who knows that user.

This ticket absorbs the former TICKET-006 (UserIdentifier Management), which no longer exists as a
separate concern.

## Technical Details

### The Registration Rule

Everything in this ticket follows from one rule:

> When a participant registers user identifier `X`: if no user with `identifier = X` exists, create
> one and link it to the participant. If one exists, add the participant to its links.
> **Never merge, never match on email, never guess.**

It is deterministic and idempotent. It replaces `checkUserIdentifier`, `TryFindMatchingUser`, the
four email-reconciliation cases, and implicit federation in the auth middleware — roughly 600 lines
of documented behaviour and its entire class of failure modes.

The trade-off is explicit and must be documented for integrators: **a participant addresses users by
the data space's global identifier, not by its own internal ID.** The optional `local_identifier`
column on the link exists to hold the participant's internal ID as a convenience; it is never a
lookup key for anyone else and never participates in matching.

### Endpoints

#### 1. Self-Register as a Data Subject — `POST /users/register` (US-UM-001)

**Auth:** `USER`

**Request Body:** none. The identifier and all attributes come from the token.

**Flow:**
1. Resolve `UserPrincipal` from the token.
2. Create the user record if absent, or return the existing one.
3. Refresh display claims (`email`, `first_name`, `last_name`) if present and changed.

**Response:** `200 OK` (existing) or `201 Created` (new) with the user representation.

This endpoint is idempotent and creates no credentials. It exists so a user-facing client can
establish the record explicitly; a user who never calls it is provisioned just-in-time on first use
anyway (TICKET-003, US-ID-005). Both paths run through the same service method.

#### 2. View My Profile — `GET /users/me` (US-UM-004)

**Auth:** `USER`

Returns the caller's identifier, optional email and name, and the list of participants they are
linked to. **No tokens are minted or returned** — the IDP issues tokens, not this service.

#### 3. Look Up a User — `GET /users/{identifier}` (US-UM-005)

**Auth:** `PARTICIPANT` or `CATALOG`

`{identifier}` is the URL-encoded global identifier.

**Flow:**
1. Resolve the user by identifier; 404 if absent.
2. For a `PARTICIPANT` caller, return the record only if the caller is linked to that user;
   otherwise **404**, not 403 — a 403 would confirm that the identifier exists.

**Note:** the pre-revision story made this a public endpoint. That leaks the existence of
identifiers to anyone who can guess an email address, so authentication is now required.

#### 4. Search Users — `POST /users/search` (US-UM-006)

**Auth:** `PARTICIPANT` or `CATALOG`

**Request Body:**
```json
{
  "identifier": "string (optional)",
  "email": "string (optional)",
  "participantIdentifier": "string (optional)"
}
```

- **Search by `identifier` is required functionality** — exact match, the primary lookup.
- **Search by `email` is optional functionality** — case-insensitive exact match, a convenience for
  operators. It may return several users or none; email is not unique and is not authoritative.
- `participantIdentifier` filters to users linked to that participant. A `PARTICIPANT` caller is
  always restricted to their own links regardless of what they pass.

At least one criterion must be supplied; an unfiltered search is rejected with 400.

This endpoint absorbs the former US-UM-009 ("does this identifier exist for this participant"),
which is just this search with both criteria set.

#### 5. Register a User — `POST /participants/me/users` (US-UM-007)

**Auth:** `PARTICIPANT`

**Request Body:**
```json
{
  "identifier": "string (required — globally unique)",
  "localIdentifier": "string (optional — the participant's own internal ID)",
  "email": "string (optional)",
  "firstName": "string (optional)",
  "lastName": "string (optional)"
}
```

**Flow:**
1. The participant is taken from the token, never from the body.
2. Apply the registration rule.
3. Create or update the `user_participants` link, storing `localIdentifier` if supplied.
4. Optional attributes are stored only when creating a new user; they never overwrite attributes on
   an existing user. One participant must not be able to rewrite another participant's view of a
   user, and IDP claims outrank participant-supplied data.

**Response:** `201 Created` (user created) or `200 OK` (existing user, link added or already present).
The body states which happened.

#### 6. Register Users in Bulk — `POST /participants/me/users/bulk` (US-UM-008)

**Auth:** `PARTICIPANT`

**Request Body:** `{ "users": [ <as above>, ... ] }`, with a configurable maximum batch size.

**Flow:** Apply the registration rule per entry. **Partial success is allowed.** Each entry gets its
own outcome; one bad entry does not roll back the batch.

**Response:** `207 Multi-Status` with a per-entry result:
```json
{
  "results": [
    { "identifier": "...", "outcome": "CREATED" },
    { "identifier": "...", "outcome": "LINKED" },
    { "identifier": "...", "outcome": "ALREADY_LINKED" },
    { "identifier": "...", "outcome": "REJECTED", "reason": "identifier must not be blank" }
  ],
  "summary": { "created": 1, "linked": 1, "alreadyLinked": 1, "rejected": 1 }
}
```

The pre-revision bulk endpoint silently skipped existing entries and reported only a count
(original bug 17.3). Per-entry outcomes replace that.

#### 7. Update a User Link — `PATCH /participants/me/users/{identifier}` (US-UM-011)

**Auth:** `PARTICIPANT`

Updates `localIdentifier` on the caller's link to that user. It cannot modify the user record
itself. 404 if the caller is not linked to that user.

#### 8. Unlink a User — `DELETE /participants/me/users/{identifier}` (US-UM-012)

**Auth:** `PARTICIPANT`

Removes the caller's link to the user. **The user record is never deleted by this operation**, and
neither are consents — a participant must not be able to erase the record of a consent by
unlinking. 404 if no link exists.

Rejected with **409 Conflict** if the caller is the provider or consumer on a consent for that user
that is still `GRANTED`; revoke or terminate first.

#### 9. Erase My User Record — `DELETE /users/me` (US-UM-014)

**Auth:** `USER`

Supports the data subject's right to erasure. The policy must be explicit and documented:

1. All `GRANTED` consents are revoked, each recording a `CONSENT_REVOKED` event.
2. All `user_participants` links are removed.
3. Personal attributes on the user row (`email`, `first_name`, `last_name`) are cleared and
   `identifier` is replaced with a non-reversible pseudonym.
4. Consents and consent events are **retained** in pseudonymised form. They are the audit trail
   proving what was consented to and when, and deleting them would destroy the evidence the service
   exists to keep.

**Response:** `200 OK` with a summary of what was revoked and anonymised.

This story has no counterpart in the pre-revision plan — a consent manager with no erasure path is a
gap, not a simplification.

### Service Layer

```
UserService
  provisionFromToken(UserPrincipal)          # JIT + explicit self-registration, idempotent
  findByIdentifier(String)
  search(UserSearchCriteria, CallerScope)
  registerForParticipant(Participant, UserRegistration)   # the registration rule
  registerBulkForParticipant(Participant, List<UserRegistration>)
  updateLink(Participant, String identifier, String localIdentifier)
  unlink(Participant, String identifier)
  erase(User)
```

`registerForParticipant` must be safe under concurrent calls for the same identifier: catch the
unique-constraint violation on `users.identifier` and re-read rather than pre-checking with a
separate query.

### Files to Create

```
src/main/java/eu/prometheusx/consentmanager/
  api/UserController.java
  api/ParticipantUserController.java       # /participants/me/users/*
  service/UserService.java
```

**Authored in the API specification, not hand-written** (TICKET-015): the request and response
models below are generated from these schemas. Controllers implement the generated interfaces.

```
api/components/schemas/User.yaml
api/components/schemas/UserRegistration.yaml
api/components/schemas/BulkUserRegistration.yaml
api/components/schemas/BulkRegistrationResult.yaml
api/components/schemas/UserSearch.yaml
api/paths/users.yaml
```

## Acceptance Criteria

1. [ ] `POST /users/register` is idempotent and creates no credentials
2. [ ] A user provisioned just-in-time and one created via `/users/register` are indistinguishable
3. [ ] `GET /users/me` returns the caller's record and participant links, and mints no tokens
4. [ ] `GET /users/{identifier}` returns 404 (not 403) to a participant not linked to that user
5. [ ] `POST /users/search` finds users by exact global identifier
6. [ ] `POST /users/search` finds users by case-insensitive email and may return several
7. [ ] `POST /users/search` rejects an unfiltered search with 400
8. [ ] A `PARTICIPANT` caller's search results are restricted to their own links
9. [ ] Registering an unknown identifier creates the user and the link, returning 201
10. [ ] Registering a known identifier adds the link only, returning 200, and leaves attributes untouched
11. [ ] Registering an already-linked identifier is a no-op reported as `ALREADY_LINKED`
12. [ ] Bulk registration reports a per-entry outcome and allows partial success
13. [ ] Bulk registration rejects a batch above the configured maximum size with 400
14. [ ] The participant is always taken from the token; a `participantIdentifier` in the body is ignored
15. [ ] Concurrent registration of the same identifier creates exactly one user
16. [ ] Unlinking removes only the link, never the user or its consents
17. [ ] Unlinking is rejected with 409 while a `GRANTED` consent exists for that pair
18. [ ] `DELETE /users/me` revokes granted consents, clears attributes, pseudonymises the identifier, and retains consent events
19. [ ] Parameterized tests cover the registration rule across all four entry states
20. [ ] Integration tests use real tokens from the Testcontainers IDP

## Known Issues Resolved by Design

| Original bug | Why it cannot recur |
|--------------|---------------------|
| 17.1 — signup leaks the password hash | No passwords exist |
| 17.2 — session key typo (`oath`) | No sessions |
| 17.3 — bulk registration skips federation | No federation; per-entry outcomes are reported |
| 17.4 — ObjectId comparison by reference | No ObjectIds |
| 17.5 — update returns stale data | Updates return the re-read entity |
| 17.6 — delete orphans references | `ON DELETE CASCADE` on the link table |
| 17.7 — JSON-LD never regenerated | User JSON-LD removed (TICKET-015) |
| 17.8 — dead route `GET /users/:userId/:consentId` | Not implemented |
| 17.9 — no unique index on email | Uniqueness is on `identifier`; email is deliberately non-unique |
| 17.10 — `user` field on UserIdentifier unused | Entity removed |
