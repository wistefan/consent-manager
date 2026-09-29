# TICKET-010: Consent Querying

## Metadata

| Field | Value |
|-------|-------|
| **Ticket ID** | TICKET-010 |
| **Title** | Consent Querying |
| **Priority** | Critical |
| **Complexity** | M (Medium) |
| **Dependencies** | TICKET-008 |
| **Related User Stories** | US-CM-011, US-CM-012, US-CM-013, US-CM-015, US-CM-016, US-CM-019, US-CM-020 |

## Description

Implement consent listing and retrieval for both data subjects and participants, consent receipt
generation, lineage traversal, and contract discovery.

**This ticket is promoted from High to Critical.** With data exchange out of scope, the Consent
Manager is no longer in the path of the transfer and can no longer refuse to forward. The query API
is now the *sole* control point: it is how a participant answers "may I process this user's data for
this purpose right now?" If it is wrong, slow, or unavailable, participants have nothing else to
consult.

## Technical Details

### Endpoints

#### 1. List My Consents — `GET /consents/me` (US-CM-011)

**Auth:** `USER`

**Query Parameters:** `page`, `size`, `status` (repeatable), `privacyNoticeId`, `format`
(`full` | `receipt`, default `full`), `includeLineage` (boolean, default `false`)

Returns a `Page<ConsentResponse>` for the caller, resolved from the token. The total count comes
from the same predicate as the result set — the original counted the entire collection and reported
it as the user's total (original bug 18.3).

#### 2. Get My Consent — `GET /consents/me/{id}` (US-CM-012)

**Auth:** `USER`

404 if unknown or not owned by the caller. Supports `format` and `includeLineage`.

#### 3. Query Consents as a Participant — `GET /consents` (US-CM-013)

**Auth:** `PARTICIPANT`

**Query Parameters:** `userIdentifier` (optional), `privacyNoticeId` (optional),
`status` (repeatable, default `GRANTED`), `page`, `size`

**This is the primary integration point of the service.** Results are restricted to consents where
the caller is provider or consumer — enforced in the query predicate, not by filtering after the
fact, so there is no window in which another participant's row is loaded.

Requesting a `userIdentifier` the caller is not linked to returns an empty page, not 403: a 403
would let a participant probe which identifiers exist.

#### 4. Check Consent Validity — `GET /consents/check` (US-CM-013)

**Auth:** `PARTICIPANT`

**Query Parameters:** `userIdentifier` (required), `privacyNoticeId` (required), `purpose`
(optional)

A purpose-built, cache-friendly answer to the one question participants actually ask before
processing:

```json
{
  "valid": true,
  "consentId": "...",
  "status": "GRANTED",
  "grantedAt": "2026-09-29T10:15:00Z",
  "purposes": [ ... ],
  "checkedAt": "2026-09-29T14:02:11Z"
}
```

`valid` is `true` only when a consent for that (user, notice) is `GRANTED` and — if `purpose` is
supplied — covers that purpose. Every other case returns `valid: false` with the reason, including
"no consent found", without disclosing whether the user exists.

Responses carry `Cache-Control: no-store`. A stale "yes" is the worst possible answer here.

#### 5. Get Consent as a Participant — `GET /consents/{id}` (US-CM-013)

**Auth:** `PARTICIPANT`

404 unless the caller is provider or consumer on that consent.

#### 6. Discover Available Contracts as a User — `GET /contracts/available` (US-CM-015)

**Auth:** `USER`

Renamed from "available exchanges". The behaviour was always contract discovery; the old name
implied a data-exchange feature that no longer exists and never described what this does.

Resolves the participants the caller is linked to, fetches bilateral and ecosystem contracts for
each from the Contract Service, deduplicates, and marks which side the caller's participants are on.

#### 7. Discover Available Contracts as a Participant — `GET /contracts/available/{role}` (US-CM-016)

**Auth:** `PARTICIPANT`

`{role}` is `provider` or `consumer`; any other value is a 400. Returns bilateral and ecosystem
contracts where the caller participates in that role.

For both discovery endpoints, an unavailable Contract Service returns 502 with no partial list —
a truncated list of contracts reads as "these are all of them" and would be actively misleading.

### Consent Receipts (US-CM-019, US-CM-020)

`format=receipt` renders the consent as an **ISO/IEC 29184** receipt, with the Kantara KI-CR-v1.1.0
structure available via `format=receipt-kantara`.

Receipt fields are read from the consent's frozen `snapshot` (TICKET-002), not from the live privacy
notice. A receipt is a record of what was agreed at the time; re-deriving it from current terms
would silently rewrite history when a notice is superseded.

Receipts are generated from typed record fields, so the original defect — `toReceipt()` referencing
a field that did not exist on the schema (original bug 18.5) — is a compile error rather than a
runtime one.

Each receipt carries a **detached JWS signature** produced by TICKET-011, verifiable against
`GET /.well-known/jwks.json`. This is what replaces the pre-revision AES+RSA payload encryption:
that scheme existed to protect consent in transit to Data Space Connectors, and with that transport
gone, what retains value is proof that a receipt is authentic and unaltered.

### Lineage Traversal (US-CM-021)

`includeLineage=true` walks `parentConsentId` upward from the requested consent, returning the chain
newest-first. The walk is bounded by a configurable maximum depth (default 20) and by a visited-ID
set, so a cycle — which the `consents_parent_not_self` check constrains but does not fully prevent
across multiple rows — terminates rather than hanging the request.

### Performance

Backed by the indexes from TICKET-002: `consents (user_id, status)`, `consents (provider_id)`,
`consents (consumer_id)`, `consents (privacy_notice_id)`. The validity check must resolve in a
single indexed lookup. Include a test asserting the query plan does not degrade to a sequential scan
on a seeded dataset.

### Files to Create

```
src/main/java/eu/prometheusx/consentmanager/
  api/ConsentQueryController.java
  api/ContractDiscoveryController.java
  service/ConsentQueryService.java
  service/ConsentReceiptService.java       # ISO/IEC 29184 and Kantara rendering
```

**Authored in the API specification, not hand-written** (TICKET-015): the request and response
models below are generated from these schemas. Controllers implement the generated interfaces.

```
api/components/schemas/ConsentValidity.yaml
api/components/schemas/AvailableContract.yaml
api/components/parameters/pagination.yaml
api/paths/contracts.yaml
```

## Acceptance Criteria

1. [ ] `GET /consents/me` returns a page of the caller's consents with a correctly filtered total
2. [ ] `GET /consents/me/{id}` returns 404 for a consent owned by another user
3. [ ] `GET /consents` restricts results in the query predicate, never by post-filtering
4. [ ] A participant querying an unlinked `userIdentifier` receives an empty page, not 403
5. [ ] `GET /consents/check` returns `valid: true` only for a `GRANTED` consent
6. [ ] `GET /consents/check` with a `purpose` outside the consent's scope returns `valid: false`
7. [ ] `GET /consents/check` reveals nothing about whether an unknown user exists
8. [ ] Validity responses carry `Cache-Control: no-store`
9. [ ] `format=receipt` renders a valid ISO/IEC 29184 receipt from the consent snapshot
10. [ ] `format=receipt-kantara` renders a valid KI-CR-v1.1.0 receipt
11. [ ] A receipt is unchanged after its privacy notice is archived and regenerated
12. [ ] Every receipt carries a JWS signature verifiable against the published JWKS
13. [ ] `includeLineage=true` returns the parent chain newest-first
14. [ ] Lineage traversal terminates at the configured depth and on a cycle
15. [ ] `GET /contracts/available/{role}` rejects a role other than `provider` or `consumer` with 400
16. [ ] An unavailable Contract Service returns 502 from discovery, never a partial list
17. [ ] Consent and validity queries use an index scan on a seeded dataset
18. [ ] Parameterized tests cover the (caller role × ownership × expected status) matrix
19. [ ] Integration tests cover pagination boundaries: empty, single page, exact multiple, partial last page

## Changes from the Pre-Revision Ticket

- Promoted to **Critical**; it is now the only consent enforcement surface.
- Added `GET /consents/check` — the explicit replacement for the removed data-exchange gate.
- "Available exchanges" renamed to contract discovery.
- Receipts are rendered from the consent snapshot and signed (JWS), replacing payload encryption.
- Participant scoping is enforced in the query predicate rather than after loading.
- Original bugs 18.3 (global count) and 18.5 (receipt field mismatch) resolved by design.
