# TICKET-001: Project Setup and Scaffolding

## Metadata

| Field | Value |
|-------|-------|
| **Ticket ID** | TICKET-001 |
| **Title** | Project Setup and Scaffolding |
| **Priority** | Critical |
| **Complexity** | L (Large) |
| **Dependencies** | None |
| **Related User Stories** | US-SM-006, US-SM-008, US-SM-009 |

## Description

Set up the foundational project structure for the reimplemented Consent Manager on the revised
technology stack: **Java 21 + Micronaut 4 + PostgreSQL 16**, built with Gradle, schema-managed with
Flyway, and tested with JUnit 5 + Testcontainers.

This ticket establishes the baseline that all other tickets build upon. It creates a running,
health-checkable, database-connected service with no domain logic.

## Technical Details

### Project Initialization

- Generate the project with Micronaut Launch (or equivalent) — **Java 21**, **Gradle (Kotlin DSL)**,
  application type `default`.
- Package root: `eu.prometheusx.consentmanager`.
- Enable the annotation processors required by Micronaut Data and Micronaut Validation.
  **Do not** enable the `micronaut-openapi` annotation processor: the API specification is authored
  by hand and the code is generated from it, not the other way around (TICKET-015).
- Configure Spotless (or equivalent) with a formatter config, plus Error Prone / NullAway if the
  team wants null-safety enforcement.

### Core Dependencies

| Dependency | Purpose |
|------------|---------|
| `micronaut-http-server-netty` | HTTP server |
| `micronaut-data-jdbc` | Repository layer (compile-time generated) |
| `micronaut-jdbc-hikari` | Connection pool |
| `postgresql` (JDBC driver) | PostgreSQL connectivity |
| `micronaut-flyway` | Versioned schema migrations |
| `micronaut-validation` | Jakarta Bean Validation on DTOs |
| `micronaut-security-jwt` | OAuth2 resource server, JWKS validation |
| `micronaut-http-client` | Declarative `@Client` for the Contract Service |
| `micronaut-cache-caffeine` | Contract and JWKS response caching |
| `micronaut-management` | Health, readiness, metrics endpoints |
| `micronaut-micrometer-registry-prometheus` | Metrics export |
| `micronaut-email-javamail` | SMTP notification transport |
| `logback-classic` + `logstash-logback-encoder` | Structured JSON logging |

### Build Plugins

| Plugin | Purpose |
|--------|---------|
| `io.micronaut.application` | Micronaut Gradle plugin |
| `org.openapitools.generator` | Generates API interfaces and models **from** `api/openapi.yaml` |
| `com.diffplug.spotless` | Formatting |

### API-First Code Generation

The OpenAPI document at `api/openapi.yaml` is the authored source of truth. The build generates
Java API interfaces and model classes from it with the OpenAPI Generator
(`java-micronaut-server` generator) before compilation.

```kotlin
openApiGenerate {
    generatorName.set("java-micronaut-server")
    inputSpec.set("$rootDir/api/openapi.yaml")
    outputDir.set("$buildDir/generated/openapi")
    apiPackage.set("eu.prometheusx.consentmanager.api.generated")
    modelPackage.set("eu.prometheusx.consentmanager.api.generated.model")
    configOptions.set(mapOf(
        "useBeanValidation" to "true",
        "serializationLibrary" to "micronaut_serde_jackson",
        "generateControllerFromExamples" to "false",
    ))
}
tasks.named("compileJava") { dependsOn("openApiGenerate") }
sourceSets["main"].java.srcDir("$buildDir/generated/openapi/src/main/java")
```

Rules:

- Generated sources live under `build/` and are **never committed**. Committing them creates a
  second source of truth that can be edited, which is exactly what this arrangement prevents.
- The generator version and every `configOptions` key is **pinned**. Generator upgrades change
  output shape, so the upgrade is a reviewed change, not something a fresh checkout picks up. Verify
  the option names against the pinned generator version — they vary between releases.
- Hand-written controllers **implement** the generated interfaces. Adding an operation to the spec
  breaks the build until someone implements it.
- DTOs are generated. No ticket hand-writes a request or response class; the `api/dto/` packages
  named in later tickets refer to schemas authored in the spec.

TICKET-015 owns the spec's structure, linting, and serving.

### Test Dependencies

| Dependency | Purpose |
|------------|---------|
| `micronaut-test-junit5` | Micronaut test integration |
| `junit-jupiter` (incl. `junit-jupiter-params`) | Test framework and parameterized tests |
| `assertj-core` | Fluent assertions |
| `testcontainers` + `testcontainers-postgresql` | Real PostgreSQL per test run |
| `testcontainers-keycloak` | Real OIDC provider for identity tests |
| `wiremock` | Contract Service stubbing |

### Package Structure to Create

```
src/main/java/eu/prometheusx/consentmanager/
  Application.java               # Micronaut entry point
  config/
    ConsentManagerConfiguration.java   # @ConfigurationProperties root
  domain/                        # (empty, created by TICKET-002)
  repository/                    # (empty, created by TICKET-002)
  service/                       # (empty, created by later tickets)
  api/
    dto/                         # (empty, created by later tickets)
  error/
    ApiError.java                # RFC 7807 Problem Details representation
    BadRequestException.java
    NotFoundException.java
    ForbiddenException.java
    UpstreamServiceException.java
    GlobalExceptionHandler.java  # stub, full implementation in TICKET-015
src/main/resources/
  application.yml                # Base configuration
  application-dev.yml            # Development overrides
  logback.xml                    # JSON to stdout
  db/migration/                  # Flyway migrations (created by TICKET-002)
src/test/java/eu/prometheusx/consentmanager/
  support/
    PostgresTestResource.java    # Testcontainers PostgreSQL fixture
docker/
  Dockerfile                     # Multi-stage, distroless JRE runtime
compose.yaml                     # App + PostgreSQL for local development
```

### Application Configuration (`application.yml`)

- `micronaut.application.name: consent-manager`
- `micronaut.server.port` bound to `PORT`, default `8080`
- `micronaut.server.context-path` bound to `API_PREFIX`, default `/v1`
- Datasource bound to `DB_URL`, `DB_USER`, `DB_PASSWORD`
- `flyway.datasources.default.enabled: true`
- `micronaut.metrics.enabled: true`, Prometheus registry on
- `endpoints.health.details-visible: ANONYMOUS` only outside production

Health and metrics endpoints are served **outside** the API context path, as are the OpenAPI
document and Swagger UI.

### Environment Configuration

| Variable | Default | Required | Description |
|----------|---------|----------|-------------|
| `PORT` | `8080` | No | HTTP listen port |
| `API_PREFIX` | `/v1` | No | Context path for all API routes |
| `DB_URL` | — | Yes | JDBC URL, e.g. `jdbc:postgresql://db:5432/consent_manager` |
| `DB_USER` | — | Yes | Database user |
| `DB_PASSWORD` | — | Yes | Database password |
| `CONSENT_MANAGER_URL` | — | Yes | Public base URL of this service |
| `CONTRACT_SERVICE_URL` | — | Yes | Contract Service base URL |
| `IDP_*` | — | Yes | Trusted IDP configuration — see TICKET-003 |
| `CONSENT_SIGNING_KEY_PATH` | — | Yes | Path to the consent receipt signing key — see TICKET-011 |
| `SMTP_*` | — | No | Notification transport — see TICKET-013 |
| `MICRONAUT_ENVIRONMENTS` | `dev` | No | Active environment profile |

There is **no** `JWT_SECRET_KEY`, `OAUTH_SECRET_KEY`, `SESSION_SECRET`, `SALT_ROUNDS` or
`X_CATALOG_CONSENT_KEY`. The service issues no tokens, stores no credentials and holds no sessions.

### Error Model

Define the exception hierarchy and an RFC 7807 Problem Details response body:

| Exception | HTTP Status | Use Case |
|-----------|-------------|----------|
| `BadRequestException` | 400 | Invalid request data or illegal state transition |
| `NotFoundException` | 404 | Resource not found |
| `ForbiddenException` | 403 | Authenticated but not permitted |
| `UpstreamServiceException` | 502 | Contract Service or other upstream failure |

`GlobalExceptionHandler` is created here as a stub returning a generic 500. The full implementation
(all exception types, constraint-violation mapping, production detail suppression) is owned by
TICKET-015.

### Logging

`logback.xml` writes **structured JSON to stdout** — no file appenders, no rotation. Log rotation
and retention are the responsibility of the container platform. Include an MDC-based correlation ID
filter so requests can be traced across log lines.

### Docker and Compose

- Multi-stage `Dockerfile`: Gradle build stage, then a distroless or `eclipse-temurin:21-jre-alpine`
  runtime stage carrying only the shadow JAR.
- `compose.yaml` starts the application plus a plain `postgres:16` container. **No replica set, no
  initialisation script** — the session store that required one is gone.
- Optionally add a GraalVM native-image build variant behind a Gradle flag; not required for MVP.

### Gradle Tasks

| Task | Purpose |
|------|---------|
| `./gradlew build` | Compile, run tests, assemble |
| `./gradlew run` | Run locally with the `dev` environment |
| `./gradlew test` | Run the test suite |
| `./gradlew spotlessApply` | Format sources |
| `./gradlew dockerBuild` | Build the container image |

## Acceptance Criteria

1. [ ] `./gradlew build` completes with no errors and no warnings from the annotation processors
2. [ ] `./gradlew run` starts the service and connects to PostgreSQL
3. [ ] Flyway runs on startup and reports the schema version (initially empty baseline)
4. [ ] `GET /health` returns `200 OK` with `UP` status including the datasource health indicator
5. [ ] `GET /health/readiness` reports `DOWN` while the database is unreachable
6. [ ] `GET /metrics` exposes Prometheus-format metrics
7. [ ] `./gradlew openApiGenerate` produces API interfaces and models from `api/openapi.yaml`
8. [ ] Generated sources are on the compile path and are excluded from version control
9. [ ] `GET /swagger-ui` serves the authored specification (a minimal document at this stage)
10. [ ] All four exception types are defined and a stub exception handler is registered
11. [ ] Logs are emitted as single-line JSON to stdout with a correlation ID field
12. [ ] `docker compose up` starts the application and PostgreSQL, and the health endpoint passes
13. [ ] `.env.sample` documents every environment variable the application reads
14. [ ] A Testcontainers PostgreSQL fixture is available to test classes and a smoke test uses it

## Notes on Removed Concerns

The following appeared in the pre-revision version of this ticket and are deliberately absent:

- **MongoDB and replica-set initialisation** — replaced by PostgreSQL with Flyway migrations. The
  replica set existed only to support the `connect-mongo` session store.
- **`express-session`, `connect-mongo`, `bcryptjs`, `jsonwebtoken` signing secrets** — the service
  is a stateless OAuth2 resource server (TICKET-003).
- **Joi** — replaced by Jakarta Bean Validation, which is declarative on DTOs.
- **Winston daily-rotate-file** — replaced by JSON to stdout.
- **`swagger-ui-express` serving a `docs/swagger.json` that nothing enforced** — the specification
  is still hand-authored, but it is now the input the code is generated from rather than a document
  maintained alongside the code and hoped to match it (TICKET-015).
- **`API_PREFIX` hardcoding risk** (original bug 13.6) — the prefix is a framework-level context
  path, so no code constructs URLs from a literal `/v1`.
- **Synchronous key loading at module import** (original bug 18.7) — keys are loaded by an
  injectable singleton with eager startup validation (TICKET-011), not by static initialisers.
