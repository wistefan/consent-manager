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
technology stack: **Java 21 + Micronaut 4 + PostgreSQL 16**, built with Maven, schema-managed with
Flyway, and tested with JUnit 5 + Testcontainers.

This ticket establishes the baseline that all other tickets build upon. It creates a running,
health-checkable, database-connected service with no domain logic.

## Technical Details

### Language Constraint

**The implementation language is Java, and only Java.** No Kotlin source set, no Kotlin plugin, no
Kotlin on the compile or test classpath.

The build is **Maven**, which means the build has no programming language at all: `pom.xml` is
declarative configuration, not a script. There is no build DSL to pick, no build logic to review,
and no second language in the repository. Gradle's Groovy and Kotlin DSLs are both out of scope by
construction.

Any code sample in these tickets is illustrative.

### Project Initialization

- Generate the project with Micronaut Launch (or equivalent) — **Java 21**, **Maven**, application
  type `default`. Commit the Maven Wrapper (`mvnw`, `.mvn/wrapper`) so the build is reproducible
  without a local Maven installation.
- Inherit from the `io.micronaut.platform:micronaut-parent` POM so dependency versions come from the
  Micronaut BOM rather than being pinned one by one.
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

### Maven Plugins

| Plugin | Purpose |
|--------|---------|
| `micronaut-maven-plugin` | Micronaut build integration, `mn:run`, container packaging |
| `maven-compiler-plugin` | Java 21, annotation processor paths |
| `openapi-generator-maven-plugin` | Generates API interfaces and models **from** `api/openapi.yaml` |
| `maven-surefire-plugin` | Unit tests |
| `maven-failsafe-plugin` | Integration tests (`*IT.java`) |
| `jacoco-maven-plugin` | Coverage, with a build-failing threshold |
| `spotless-maven-plugin` | Formatting |

Annotation processors (Micronaut Data, Micronaut Validation, Micronaut Inject) are declared in
`maven-compiler-plugin`'s `annotationProcessorPaths` rather than as ordinary dependencies, so they
stay off the runtime classpath.

### API-First Code Generation

The OpenAPI document at `api/openapi.yaml` is the authored source of truth. The build generates
Java API interfaces and model classes from it with the OpenAPI Generator
(`java-micronaut-server` generator) before compilation.

```xml
<plugin>
  <groupId>org.openapitools</groupId>
  <artifactId>openapi-generator-maven-plugin</artifactId>
  <version>${openapi.generator.version}</version>
  <executions>
    <execution>
      <goals><goal>generate</goal></goals>
      <configuration>
        <generatorName>java-micronaut-server</generatorName>
        <inputSpec>${project.basedir}/api/openapi.yaml</inputSpec>
        <output>${project.build.directory}/generated-sources/openapi</output>
        <apiPackage>eu.prometheusx.consentmanager.api.generated</apiPackage>
        <modelPackage>eu.prometheusx.consentmanager.api.generated.model</modelPackage>
        <configOptions>
          <useBeanValidation>true</useBeanValidation>
          <serializationLibrary>micronaut_serde_jackson</serializationLibrary>
          <generateControllerFromExamples>false</generateControllerFromExamples>
        </configOptions>
      </configuration>
    </execution>
  </executions>
</plugin>
```

The execution binds to `generate-sources`, which runs before `compile` in the default lifecycle —
the ordering is a property of the lifecycle rather than something the build has to declare. The
plugin registers its output directory as a compile source root itself, so no `build-helper` step is
needed.

Rules:

- Generated sources live under `target/generated-sources/` and are **never committed**. Committing
  them creates a second source of truth that can be edited, which is exactly what this arrangement
  prevents.
- The generator version and every `configOptions` key is **pinned** via a `${openapi.generator.version}`
  property. Generator upgrades change output shape, so the upgrade is a reviewed change, not
  something a fresh checkout picks up. Verify the option names against the pinned generator version
  — they vary between releases.
- Hand-written controllers **implement** the generated interfaces. Adding an operation to the spec
  breaks the build until someone implements it.
- DTOs are generated. No ticket hand-writes a request or response class; the schema files listed in
  later tickets are the authored source they are generated from.

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

- Multi-stage `Dockerfile`: Maven build stage, then a distroless or `eclipse-temurin:21-jre-alpine`
  runtime stage carrying only the packaged JAR.
- `compose.yaml` starts the application plus a plain `postgres:16` container. **No replica set, no
  initialisation script** — the session store that required one is gone.
- Optionally add a GraalVM native-image variant via a Maven profile; not required for MVP.

### Maven Commands

| Command | Purpose |
|---------|---------|
| `./mvnw verify` | Generate, compile, run unit and integration tests, package |
| `./mvnw mn:run` | Run locally with the `dev` environment |
| `./mvnw test` | Unit tests only (Surefire) |
| `./mvnw verify -DskipUTs` | Integration tests only (Failsafe) |
| `./mvnw spotless:apply` | Format sources |
| `./mvnw package -Dpackaging=docker` | Build the container image |

## Acceptance Criteria

1. [ ] `./mvnw verify` completes with no errors and no warnings from the annotation processors
2. [ ] The build is Maven; no Gradle file, build DSL, or Kotlin plugin exists in the project
3. [ ] The Maven Wrapper is committed and the build succeeds without a local Maven installation
4. [ ] `./mvnw mn:run` starts the service and connects to PostgreSQL
5. [ ] Flyway runs on startup and reports the schema version (initially empty baseline)
6. [ ] `GET /health` returns `200 OK` with `UP` status including the datasource health indicator
7. [ ] `GET /health/readiness` reports `DOWN` while the database is unreachable
8. [ ] `GET /metrics` exposes Prometheus-format metrics
9. [ ] `./mvnw generate-sources` produces API interfaces and models from `api/openapi.yaml`
10. [ ] Generated sources are on the compile path and are excluded from version control
11. [ ] `GET /swagger-ui` serves the authored specification (a minimal document at this stage)
12. [ ] All four exception types are defined and a stub exception handler is registered
13. [ ] Logs are emitted as single-line JSON to stdout with a correlation ID field
14. [ ] `docker compose up` starts the application and PostgreSQL, and the health endpoint passes
15. [ ] `.env.sample` documents every environment variable the application reads
16. [ ] A Testcontainers PostgreSQL fixture is available to test classes and a smoke test uses it

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
