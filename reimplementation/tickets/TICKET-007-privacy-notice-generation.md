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

Generate privacy notices from contracts fetched from the Contract Service, archive superseded
notices, and serve notices with external data resolved.

Privacy notices are the bridge between contracts (owned by the Contract Service) and consents (owned
by this service): they describe what data will be processed, for what purpose, and under what terms.

**The Contract Service remains the largest external dependency of the Consent Manager.** Removing
data exchange from scope did not change that — consent is still meaningless without the contract
that defines what is being consented to.

## Technical Details

### API Change: Identifiers Instead of Base64 URLs

The pre-revision API passed provider and consumer as **base64-encoded self-description URLs** in
path and body parameters, and resolved participants by matching `selfDescriptionURL`.

That is replaced throughout: parties are named by their **global participant identifier**, and
resolved with `ParticipantRepository.findByIdentifier`. Identifiers travel as ordinary URL-encoded
strings. Nothing in the service base64-encodes a parameter any more.

### Endpoints

#### 1. Generate a Privacy Notice Between Parties — `POST /privacy-notices` (US-CM-001)

**Auth:** `PARTICIPANT`

**Request Body:**
```json
{
  "contractUri": "string (required)",
  "dataProvider": "string (required — global participant identifier)",
  "dataConsumer": "string (required — global participant identifier)"
}
```

**Flow:**
1. Resolve both participants by identifier; 404 if either is unknown.
2. Verify the caller is one of the two named parties; otherwise **403**.
3. Fetch the contract from the Contract Service.
4. Look for an active (`archived_at IS NULL`) notice for `(contractUri, providerId)`:
   - identical contract content → return it, `200 OK`, no duplicate created;
   - contract has changed → archive the existing notice, then create a new one.
5. Extract the notice content from the contract (see below) and persist it as the `payload` JSONB.

**Response:** `201 Created` (new) or `200 OK` (existing).

The partial unique index from TICKET-002 enforces "at most one active notice per contract and
provider" at the database level, so a race between two concurrent generations resolves to one
active notice rather than two.

#### 2. Generate a Privacy Notice by Contract URI — `POST /privacy-notices/by-contract` (US-CM-022)

**Auth:** `PARTICIPANT`

**Request Body:** `{ "contractUri": "string (required)" }`

Fetches the contract and derives the provider and consumer from the contract itself rather than
requiring the caller to name them. Same duplicate and archival handling as above. The caller must
still be one of the parties named in the fetched contract.

#### 3. Retrieve a Privacy Notice — `GET /privacy-notices/{id}` (US-CM-002)

**Auth:** `USER`, `PARTICIPANT`, or `CATALOG`

**Query Parameters:** `resolve` (boolean, default `true`) — resolve external references.

**Flow:**
1. Load the notice; 404 if unknown.
2. When `resolve=true`, populate from the Contract Service and the referenced self-descriptions:
   contract title and terms, data resource descriptions, purpose descriptions, recipient details.
3. Return the notice with a `resolution` block reporting what was resolved and what was not.

**Response:** `200 OK`

Archived notices remain retrievable by ID — a consent granted against one must stay readable — and
carry `archivedAt` in the response.

### Contract Service Client

A declarative Micronaut `@Client`, replacing the ad-hoc Axios calls:

```java
@Client("${consent-manager.contract-service.url}")
public interface ContractServiceClient {
    @Get("/contracts/{contractId}")
    Optional<ContractResponse> fetchContract(String contractId);

    @Get("/bilaterals/{contractId}")
    Optional<ContractResponse> fetchBilateralContract(String contractId);

    @Get("/contracts/for/{participantIdentifier}")
    List<ContractResponse> fetchContractsFor(String participantIdentifier, @QueryValue boolean hasSigned);

    @Get("/bilaterals/for/{participantIdentifier}")
    List<ContractResponse> fetchBilateralsFor(String participantIdentifier, @QueryValue boolean hasSigned);
}
```

Configuration: connect and read timeouts, a bounded retry with backoff on 5xx and timeouts (never on
4xx), and a Caffeine cache with a configurable TTL on contract reads — replacing
`axios-cache-interceptor`.

### Content Extraction

From the contract response, derive the notice payload:

| Notice field | Source |
|--------------|--------|
| `title` | Contract title |
| `purposes` | Service offering policies |
| `categoriesOfData`, `data` | Data resource descriptions referenced by the offerings |
| `recipients` | Contract parties |
| `controllerDetails` | Provider participant record |
| `retentionPeriod`, `piiPrincipalRights`, `internationalTransfers`, `withdrawalOfConsent` | Contract terms |

### Partial Resolution, Never Partial Failure

Resolving a notice fans out to several external documents. A failure to fetch one of them must not
fail the request: return the notice with that section unresolved and a machine-readable note saying
so.

```json
{
  "id": "...",
  "contractUri": "...",
  "purposes": [ ... ],
  "resolution": {
    "contract": "RESOLVED",
    "dataResources": "UNAVAILABLE",
    "purposes": "RESOLVED",
    "recipients": "RESOLVED"
  }
}
```

A hard failure is warranted in exactly one place: **generation** cannot proceed without the contract
itself, so an unreachable Contract Service there returns `502` via `UpstreamServiceException`.

### Files to Create

```
src/main/java/eu/prometheusx/consentmanager/
  api/PrivacyNoticeController.java
  service/PrivacyNoticeService.java
  service/ContractContentExtractor.java
  client/ContractServiceClient.java
  client/dto/ContractResponse.java
```

**Authored in the API specification, not hand-written** (TICKET-015): the request and response
models below are generated from these schemas. Controllers implement the generated interfaces.

```
api/components/schemas/PrivacyNotice.yaml
api/components/schemas/GeneratePrivacyNotice.yaml
api/paths/privacy-notices.yaml
```

## Acceptance Criteria

1. [ ] `POST /privacy-notices` generates a notice from a contract for two named participants
2. [ ] Parties are named by global identifier; no parameter anywhere is base64-encoded
3. [ ] A caller that is neither provider nor consumer receives 403
4. [ ] Generating twice for an unchanged contract returns the existing notice with 200
5. [ ] Generating after a contract change archives the old notice and creates a new one
6. [ ] Concurrent generation for the same contract and provider yields exactly one active notice
7. [ ] `POST /privacy-notices/by-contract` derives both parties from the contract
8. [ ] `GET /privacy-notices/{id}` resolves contract, data resources, purposes and recipients
9. [ ] An unavailable external document yields a partial response with `UNAVAILABLE`, not a 5xx
10. [ ] An unreachable Contract Service during generation returns 502
11. [ ] Archived notices remain retrievable by ID and report `archivedAt`
12. [ ] Contract reads are cached and a repeated generation makes no second upstream call
13. [ ] Retries apply to 5xx and timeouts only, never to 4xx
14. [ ] Integration tests stub the Contract Service with WireMock
15. [ ] Parameterized tests cover the extraction matrix across contract shapes, including missing optional sections

## Changes from the Pre-Revision Ticket

- Participants are resolved by **global identifier**, not by matching `selfDescriptionURL`, and
  parameters are no longer base64-encoded.
- The Contract Service client is a declarative `@Client` with typed responses, timeouts, bounded
  retry and a Caffeine cache, replacing raw Axios calls.
- Partial resolution is now a specified response shape rather than an unhandled failure mode.
- Duplicate prevention is backed by a database constraint, not only by an application-level check.
