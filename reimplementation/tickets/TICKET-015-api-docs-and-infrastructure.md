# TICKET-015: API Docs and Cross-Cutting Infrastructure

## Metadata

| Field | Value |
|-------|-------|
| **Ticket ID** | TICKET-015 |
| **Title** | API Docs and Cross-Cutting Infrastructure |
| **Priority** | High |
| **Complexity** | M (Medium) |
| **Dependencies** | TICKET-001 |
| **Related User Stories** | US-SM-005, US-SM-006, US-SM-008, US-SM-009, US-SM-011, US-ID-003 |

## Description

Build the cross-cutting concerns that every module depends on: generated API documentation, health
and readiness probes, structured logging, error handling, and the Contract Service test double.

**Relationship to TICKET-001:** TICKET-001 creates the exception types and a stub exception handler,
a baseline `logback.xml`, and working health endpoints. This ticket replaces the stubs with the full
implementations. It does not redefine the exception types.

## Technical Details

### 1. API Specification — Authored First, Code Generated From It

**The OpenAPI document is written by hand and the code is generated from it.** The specification is
the design artefact and the contract; the Java API interfaces and model classes are its output.

This is the opposite of the annotation-driven arrangement, and the reason is that the specification
is the thing integrators build against. Deriving it from whatever the code happens to do makes the
contract an afterthought — it can only be reviewed after the implementation exists, and every
implementation detail that leaks into an annotation leaks into the published contract. Authoring it
first means the contract is reviewable before anyone writes a handler, and the implementation cannot
quietly diverge from it because it will not compile.

#### Source Layout

```
api/
  openapi.yaml                 # root document
  paths/
    consents.yaml
    consent-requests.yaml
    privacy-notices.yaml
    users.yaml
    participants.yaml
    contracts.yaml
    notifications.yaml
  components/
    schemas/                   # request and response schemas
    responses/                 # shared error responses
    parameters/                # pagination, format, filters
    security.yaml              # securitySchemes, the token contract
  .spectral.yaml               # lint ruleset
```

Split by resource with `$ref`, bundled to a single document for publication. A single flat file for
an API this size becomes unreviewable — a diff touching one endpoint should not land in the middle
of a two-thousand-line file.

#### Generation

The OpenAPI Generator (`java-micronaut-server`) runs before `compileJava` and produces:

- **API interfaces** — one per tag, with typed parameters and return types
- **Model classes** — with Jakarta Validation annotations derived from the schema constraints

Controllers are hand-written and `implement` the generated interfaces. Consequences worth stating
plainly:

- Adding an operation to the spec **breaks the build** until it is implemented.
- Changing a parameter or response type in the spec **breaks the build** at every affected call
  site.
- Removing an operation from the spec leaves an `@Override` that no longer overrides anything —
  also a build break.

Drift in signatures is therefore not a thing that can be reviewed for, forgotten, or linted. It is a
compile error.

#### Validation Constraints Come From the Spec

`required`, `minLength`, `maxLength`, `pattern`, `format`, `minimum`/`maximum` and `enum` in the
schemas generate the corresponding Bean Validation annotations on the model classes. The request
constraints named in TICKET-004, TICKET-005 and TICKET-008 are authored **once**, in the schema, and
are simultaneously the documented contract and the enforced rule. They cannot disagree.

Hand-written constraints remain only for rules a JSON Schema cannot express — a purpose being a
subset of the notice's purposes, for instance. Those live in the service layer, not on the DTO.

#### The Error Contract Is Declared Once

RFC 7807 Problem Details is a shared schema (`components/schemas/ProblemDetail`) with shared
responses (`components/responses/BadRequest`, `Unauthorized`, `Forbidden`, `NotFound`, `Conflict`,
`UpstreamError`). Every operation `$ref`s them. An endpoint cannot document an ad-hoc error shape,
and adding a status to the shared set documents it everywhere at once.

#### The Token Contract Lives in `components/security.yaml`

The token contract from TICKET-003 §3 is authored here as `securitySchemes` plus a description
enumerating every required and optional claim for user and participant tokens, and the role each
operation needs. Integrators cannot configure their IDP without it, and this is where they will look
for it.

Each operation declares its `security` requirement, matching the `@Secured` annotation on the
implementing controller. The generator does not enforce that pairing, so it is an explicit review
obligation and is covered by the authorization test matrix in TICKET-019.

#### Linting and Compatibility

- **Spectral** runs in CI against `.spectral.yaml`: every operation has an `operationId`, a summary,
  a `security` entry and documented error responses; every schema property has a description; naming
  is consistent. A lint failure fails the build.
- **Breaking-change detection** (`oasdiff` or equivalent) compares the spec against the last
  released version. A breaking change requires an explicit acknowledgement in the PR rather than
  passing silently.

#### Serving

The bundled document is served as a static resource at `GET /openapi.yaml`, with Swagger UI at
`GET /swagger-ui`, both outside the API context path. What is served is the authored file — not a
re-derivation of it — so what integrators read is exactly what the code was generated from.

### 2. Health and Readiness (US-SM-006)

Two distinct endpoints, served outside the API context path:

| Endpoint | Purpose | Contents |
|----------|---------|----------|
| `GET /health/liveness` | Is the process alive? | Process only. **No dependency checks.** |
| `GET /health/readiness` | Should it receive traffic? | Datasource connectivity, Flyway migration state, IDP discovery/JWKS availability |

The distinction matters operationally: a liveness probe that checks dependencies makes an IDP or
database blip restart every pod, turning a partial outage into a total one. Liveness must fail only
when the process itself is unrecoverable.

`endpoints.health.details-visible` is `ANONYMOUS` outside production and `AUTHENTICATED` in
production — dependency hostnames and failure messages are reconnaissance material.

Prometheus metrics at `GET /metrics` via Micrometer, including JVM, HTTP server, HikariCP and
custom counters for consent state transitions and token validation failures.

### 3. Structured Logging (US-SM-009)

Logback emitting **single-line JSON to stdout**. No file appenders, no rotation, no retention
configuration — that is the container platform's responsibility, and writing rotated files inside a
container is how log data gets lost when the pod is replaced.

Every line carries: timestamp, level, logger, message, thread, and a correlation ID from MDC,
propagated from an inbound `X-Request-ID` or generated per request and echoed in the response.

**What must never be logged**, enforced by a test that scans emitted output:

- Bearer tokens, in whole or in part, including in URLs and exception messages
- Private key material
- User identifiers, email addresses and names at `INFO` or above

Personal identifiers appear at `DEBUG` only, which is off by default in production. This is a
service whose entire purpose is handling personal data lawfully; leaking that data into logs
undermines the thing it exists to do. Where an identifier is needed for correlation at `INFO`, log a
stable hash of it instead.

### 4. Request Validation

Jakarta Bean Validation on request models. The annotations are **generated from the schema
constraints** in `api/openapi.yaml` (Section 1), not hand-written: `required`, `minLength`,
`pattern`, `format: email` and `enum` become `@NotNull`, `@Size`, `@Pattern`, `@Email` and a typed
enum. Micronaut validates before the handler runs, so a handler never receives a malformed payload,
and the documented constraint and the enforced constraint are the same declaration.

This is why **US-SM-007 no longer exists as a user story**. It described hand-written Joi
middleware, a work item that only existed because Express has no validation layer. Here the
constraints are authored in the specification and enforced by the framework; what remains is one
exception handler. The requirement survives as acceptance criteria below; the story does not.

### 5. Error Handling (US-SM-008) — RFC 7807

All errors return `application/problem+json`:

```json
{
  "type": "https://consent-manager.example/problems/invalid-state-transition",
  "title": "Invalid state transition",
  "status": 400,
  "detail": "Consent is REVOKED and cannot transition to GRANTED",
  "instance": "/v1/consents/3f9a.../resume",
  "correlationId": "01J8X...",
  "errors": [ { "field": "legalName", "message": "must not be blank" } ]
}
```

| Exception | Status |
|-----------|--------|
| `BadRequestException` | 400 |
| `ConstraintViolationException` | 400, with a per-field `errors` array |
| `AuthenticationException` | 401 |
| `ForbiddenException` | 403 |
| `NotFoundException` | 404 |
| Conflict (duplicate, illegal concurrent state) | 409 |
| `UpstreamServiceException` | 502 |
| Anything else | 500 |

Every handler is registered — the original handled two of five error classes and let the rest fall
through as 500s (original bug 13.2). A test asserts that each exception type maps to its status.

In production, 500 responses carry a generic `detail` and the correlation ID; the exception message
and stack trace go to the log only. Outside production, the real message is included.

The correlation ID appears in both the response and the log line, so a user reporting an error gives
operators the exact log entry.

### 6. Self-Descriptions (US-SM-011)

Reduced to storage. `Participant.selfDescriptionUri` is persisted and returned as an **opaque
string**: not fetched, not resolved, not validated, not interpreted.

**User JSON-LD generation is removed.** It existed to produce a schema.org `Person` document at
signup; there is no signup, and user identity is external and often deliberately pseudonymous — a
DID or a PID-derived hash. Generating a self-description for such a subject would add nothing and
would invite personal data into a document nobody consumes.

This removes original bug 13.6 (JSON-LD hardcoding the API prefix) along with the generator.

### 7. Contract Service Test Double

The pre-revision design mounted a simulated contract router **inside the application** in
development mode. That is why simulated routes escaped the global error handler (original bug 13.1),
and it puts test scaffolding in the production artefact.

Replaced by **WireMock stubs in the test source set** (TICKET-019). They are unreachable from a
production build, and nothing about the running service changes between environments. A stub set for
local manual development ships as a standalone WireMock mapping directory referenced by
`compose.yaml`, not as application code.

This also removes original bug 13.5 (simulated filtering checked `assignee` for both provider and
consumer) by removing the code that had the bug.

### Files to Create

```
api/
  openapi.yaml                           # authored root document
  paths/*.yaml
  components/schemas/*.yaml
  components/responses/*.yaml
  components/parameters/*.yaml
  components/security.yaml               # securitySchemes + token contract
  .spectral.yaml
src/main/java/eu/prometheusx/consentmanager/
  error/GlobalExceptionHandler.java      # replaces the TICKET-001 stub
  error/ProblemDetail.java
  observability/CorrelationIdFilter.java
  observability/ReadinessHealthIndicator.java
  observability/ConsentMetrics.java
src/main/resources/
  logback.xml                            # JSON to stdout, full configuration
docker/wiremock/mappings/                # local development contract stubs
```

## Acceptance Criteria

1. [ ] `api/openapi.yaml` is the authored source of truth; no code is annotated to produce a spec
2. [ ] API interfaces and models are generated from the spec before compilation
3. [ ] Generated sources are never committed and are excluded from version control
4. [ ] Every controller implements a generated interface
5. [ ] Adding an operation to the spec fails the build until it is implemented
6. [ ] Changing a response type in the spec fails the build at every affected call site
7. [ ] Validation annotations on request models originate from schema constraints, not hand-written code
8. [ ] All error responses `$ref` the shared `ProblemDetail` schema; no operation declares its own error shape
9. [ ] The spec documents the token contract, the required role per operation, and the 401/403 distinction
10. [ ] Spectral lint passes and a violation fails the build
11. [ ] A breaking spec change is detected and requires explicit acknowledgement
12. [ ] The bundled spec is served at `GET /openapi.yaml` and rendered at `GET /swagger-ui`
13. [ ] The served document is byte-identical to the bundled authored spec
14. [ ] `GET /health/liveness` checks the process only and never fails on a dependency outage
15. [ ] `GET /health/readiness` reports `DOWN` when the database or IDP discovery is unavailable
16. [ ] Health details are hidden from anonymous callers in production
17. [ ] `GET /metrics` exposes JVM, HTTP, pool and consent-transition metrics
18. [ ] Logs are single-line JSON on stdout with a correlation ID on every line
19. [ ] A test scanning emitted logs finds no token, key material, or personal identifier above DEBUG
20. [ ] The correlation ID is taken from `X-Request-ID` when present and echoed in the response
21. [ ] Bean Validation rejects malformed payloads with 400 before the handler runs
22. [ ] Validation errors list every offending field, not only the first
23. [ ] Every exception type maps to its documented status; none falls through to 500
24. [ ] Production 500 responses expose no exception message or stack trace
25. [ ] Every error response carries the correlation ID
26. [ ] No simulated or test-only route exists in the production artefact
27. [ ] Parameterized tests cover the exception-to-status mapping

## Changes from the Pre-Revision Ticket

- **API-first**: the OpenAPI document is authored and the code generated from it, replacing both
  the pre-revision hand-maintained `swagger.json` and the annotation-derived spec proposed earlier
  in this revision.
- Validation constraints and the error contract are declared once, in the schema.
- Liveness and readiness split; readiness includes IDP availability.
- Winston with daily-rotate-file replaced by Logback JSON to stdout.
- Joi validation middleware replaced by Bean Validation; **US-SM-007 removed as a story**.
- Ad-hoc error shape replaced by RFC 7807 Problem Details with correlation IDs.
- User JSON-LD generation removed; participant self-description reduced to an opaque stored URI.
- Simulated contract router moved out of the application into WireMock stubs.
