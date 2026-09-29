# TICKET-018: Deployment Configuration

## Metadata

| Field | Value |
|-------|-------|
| **Ticket ID** | TICKET-018 |
| **Title** | Deployment Configuration |
| **Priority** | Medium |
| **Complexity** | M (Medium) |
| **Dependencies** | TICKET-001 |
| **Related User Stories** | (infrastructure — no direct user stories) |

## Description

Package and deploy the service: container image, local Compose environment, Helm chart for
Kubernetes, and configuration and secret management.

## Technical Details

### Container Image

Multi-stage build:

**Stage 1 — build:** `eclipse-temurin:21-jdk` with the Gradle wrapper, dependencies resolved in a
cached layer before sources are copied, producing the shadow JAR.

**Stage 2 — runtime:** `eclipse-temurin:21-jre-alpine` or a distroless Java 21 base, carrying only
the JAR. Runs as a **non-root** user, with a read-only root filesystem and a writable `/tmp`.

JVM flags: `-XX:MaxRAMPercentage=75 -XX:+UseG1GC -XX:+ExitOnOutOfMemoryError`. The percentage form
respects the container memory limit, which a fixed `-Xmx` does not.

An optional GraalVM native-image variant behind a Gradle flag gives faster startup and a smaller
footprint; it is not required for MVP and must not be the default, since native-image reflection
configuration is an ongoing maintenance cost.

### Local Development — `compose.yaml`

Three services:

| Service | Image | Purpose |
|---------|-------|---------|
| `consent-manager` | built locally | The application |
| `postgres` | `postgres:16-alpine` | Database |
| `keycloak` | `quay.io/keycloak/keycloak` | A working IDP, with a realm import defining the three roles and a test user and client |

Plus an optional `wiremock` service serving the Contract Service stubs (TICKET-015).

**PostgreSQL needs no special configuration** — no replica set, no initialisation script. The
pre-revision setup required a MongoDB replica set solely because `connect-mongo` needed transactions
for the session store; with sessions gone, so is that requirement.

Including Keycloak is deliberate. The service cannot serve a single authenticated request without an
IDP, so a Compose file without one produces an environment where nothing works and the first-run
experience is a 401.

### Helm Chart

| Template | Purpose |
|----------|---------|
| `deployment.yaml` | Deployment with replicas, resources, probes, security context |
| `service.yaml` | ClusterIP service |
| `ingress.yaml` | Optional ingress |
| `configmap.yaml` | Non-sensitive configuration, including the IDP registry |
| `secret.yaml` | Database credentials and SMTP credentials |
| `hpa.yaml` | Optional autoscaler |
| `servicemonitor.yaml` | Optional Prometheus scrape configuration |

#### Probes

```yaml
livenessProbe:
  httpGet: { path: /health/liveness, port: http }
  initialDelaySeconds: 30
  periodSeconds: 30
  failureThreshold: 3

readinessProbe:
  httpGet: { path: /health/readiness, port: http }
  initialDelaySeconds: 10
  periodSeconds: 10
  failureThreshold: 3

startupProbe:
  httpGet: { path: /health/liveness, port: http }
  periodSeconds: 5
  failureThreshold: 30
```

The startup probe covers JVM start and Flyway migration without forcing a long
`initialDelaySeconds` on the liveness probe, which would delay recovery from a genuine hang for the
whole life of the pod.

Liveness must point at `/health/liveness`, never at `/health/readiness` — a readiness endpoint
checks dependencies, so wiring it to liveness turns a database blip into a restart loop.

#### Signing Keys

The consent receipt signing keys (TICKET-011) are mounted from a Kubernetes Secret as files:

```yaml
volumes:
  - name: signing-keys
    secret:
      secretName: {{ .Release.Name }}-signing-keys
      defaultMode: 0400
volumeMounts:
  - name: signing-keys
    mountPath: /etc/consent-manager/keys
    readOnly: true
```

Keys are **never** baked into the image, passed as environment variables, or committed. Document the
key rotation procedure: add the new key to the Secret, add it to the configured key list, point
`active-key-id` at it, roll the deployment, and retire the old key only once no receipt requiring it
is still in circulation.

#### Database Migration on Startup

Flyway runs at application startup. With multiple replicas, concurrent migration attempts are
serialised by Flyway's lock, so the deployment is safe as-is. For larger schemas, an init container
or a pre-upgrade Helm hook running migrations once is the cleaner option — document both and default
to startup migration.

Migrations must be backward-compatible for the duration of a rolling update, since old and new pods
run against the same schema. Expand-then-contract: add columns and backfill in one release, remove
the old ones in a later one.

### Configuration and Secrets

| Kind | Source |
|------|--------|
| Non-sensitive (ports, URLs, IDP registry, cache TTLs) | ConfigMap |
| Database credentials, SMTP credentials | Secret |
| Signing keys | Secret, mounted as files |

The IDP registry lives in the ConfigMap: it contains issuer URLs, claim names and role mappings —
public configuration, not secrets. **Changing it is a deliberate redeploy**, which is the intended
property; the trust list is the service's security boundary (TICKET-003) and should not be mutable
at runtime.

`.env.sample` documents every variable the application reads, matching the table in TICKET-001. It
contains **no** `JWT_SECRET_KEY`, `OAUTH_SECRET_KEY`, `SESSION_SECRET`, `SALT_ROUNDS` or
`X_CATALOG_CONSENT_KEY` — none of them exist any more.

### Files to Create

```
docker/
  Dockerfile
  wiremock/mappings/
compose.yaml
keycloak/realm-export.json
charts/consent-manager/
  Chart.yaml
  values.yaml
  templates/
    deployment.yaml
    service.yaml
    ingress.yaml
    configmap.yaml
    secret.yaml
    hpa.yaml
    servicemonitor.yaml
    _helpers.tpl
.env.sample
.dockerignore
```

## Acceptance Criteria

1. [ ] The image builds and runs as a non-root user with a read-only root filesystem
2. [ ] Heap sizing follows the container memory limit rather than a fixed `-Xmx`
3. [ ] `docker compose up` starts the application, PostgreSQL and Keycloak, and the readiness probe passes
4. [ ] The Keycloak realm import provides the three roles plus a working test user and client
5. [ ] A consent can be granted end to end in the Compose environment using a real token
6. [ ] `helm install` succeeds with default values and `helm lint` is clean
7. [ ] Liveness points at `/health/liveness` and readiness at `/health/readiness`
8. [ ] The startup probe tolerates JVM start plus migration without inflating the liveness delay
9. [ ] A database outage flips readiness to `DOWN` without triggering restarts
10. [ ] Signing keys are mounted from a Secret with mode `0400` and appear in no image layer
11. [ ] Database and SMTP credentials come from a Secret, never a ConfigMap
12. [ ] The IDP registry is rendered into the ConfigMap from Helm values
13. [ ] Flyway migrations apply exactly once across a multi-replica rollout
14. [ ] `.env.sample` documents every variable and contains no obsolete authentication settings
15. [ ] `.dockerignore` excludes build output, keys, `.env` files and test resources
16. [ ] The key rotation procedure is documented in the chart README

## Changes from the Pre-Revision Ticket

- Node image replaced by a JVM image; optional native-image variant.
- MongoDB with replica-set initialisation replaced by plain PostgreSQL.
- Keycloak added to the local environment — the service is unusable without an IDP.
- Liveness and readiness split, with a startup probe added.
- Flyway migration behaviour under rolling updates specified.
- Signing keys mounted from a Secret, with a documented rotation procedure.
- Terraform files dropped: the pre-revision `terraform/` directory held only provider boilerplate
  with no resources. Infrastructure provisioning is deployment-environment specific and is better
  owned where it is deployed than carried as an empty placeholder here.
