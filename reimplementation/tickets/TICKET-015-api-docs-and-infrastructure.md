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

### 1. API Documentation — Generated, Not Maintained

`micronaut-openapi` generates the OpenAPI 3.1 specification from controller and DTO annotations at
**build time**. The spec is an output of compilation, so it cannot drift from the code — the
pre-revision plan maintained `docs/swagger.json` by hand, which guarantees drift.

- Spec written to `META-INF/swagger/consent-manager-{version}.yml` during build
- Served at `GET /swagger/consent-manager-{version}.yml`
- Swagger UI at `GET /swagger-ui`, outside the API context path
- A build check fails if a public endpoint lacks `@Operation` documentation

**The token contract (TICKET-003 §3) is published here**, as an `securitySchemes` entry plus a
description block enumerating every required and optional claim for user and participant tokens.
Integrators cannot configure their IDP without it, so it belongs in the generated documentation and
not only in a repository markdown file.

Document per endpoint: required role, the 401-versus-403 distinction, pagination parameters, and the
error shape.

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

Jakarta Bean Validation on request DTOs — `@NotBlank`, `@Email`, `@Pattern`, `@Size`, `@Valid` on
nested types. Micronaut validates before the handler runs, so a handler never receives a malformed
payload.

This is why **US-SM-007 no longer exists as a user story**. It described hand-written Joi middleware,
a work item that only existed because Express has no validation layer. Here it is a framework
capability plus one exception handler. The requirement survives as acceptance criteria below; the
story does not.

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

1. [ ] The OpenAPI spec is generated at build time and served with Swagger UI
2. [ ] The build fails if a public endpoint lacks `@Operation` documentation
3. [ ] The published spec documents the token contract, required roles, and the 401/403 distinction
4. [ ] `GET /health/liveness` checks the process only and never fails on a dependency outage
5. [ ] `GET /health/readiness` reports `DOWN` when the database or IDP discovery is unavailable
6. [ ] Health details are hidden from anonymous callers in production
7. [ ] `GET /metrics` exposes JVM, HTTP, pool and consent-transition metrics
8. [ ] Logs are single-line JSON on stdout with a correlation ID on every line
9. [ ] A test scanning emitted logs finds no token, key material, or personal identifier above DEBUG
10. [ ] The correlation ID is taken from `X-Request-ID` when present and echoed in the response
11. [ ] Bean Validation rejects malformed payloads with 400 before the handler runs
12. [ ] Validation errors list every offending field, not only the first
13. [ ] Every exception type maps to its documented status; none falls through to 500
14. [ ] Production 500 responses expose no exception message or stack trace
15. [ ] Every error response carries the correlation ID
16. [ ] No simulated or test-only route exists in the production artefact
17. [ ] Parameterized tests cover the exception-to-status mapping

## Changes from the Pre-Revision Ticket

- OpenAPI spec generated at build time, replacing a hand-maintained `swagger.json`.
- Liveness and readiness split; readiness includes IDP availability.
- Winston with daily-rotate-file replaced by Logback JSON to stdout.
- Joi validation middleware replaced by Bean Validation; **US-SM-007 removed as a story**.
- Ad-hoc error shape replaced by RFC 7807 Problem Details with correlation IDs.
- User JSON-LD generation removed; participant self-description reduced to an opaque stored URI.
- Simulated contract router moved out of the application into WireMock stubs.
