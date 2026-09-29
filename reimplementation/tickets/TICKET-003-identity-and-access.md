# TICKET-003: Identity and Access

## Metadata

| Field | Value |
|-------|-------|
| **Ticket ID** | TICKET-003 |
| **Title** | Identity and Access |
| **Priority** | Critical |
| **Complexity** | L (Large) |
| **Dependencies** | TICKET-001, TICKET-002 |
| **Related User Stories** | US-ID-001, US-ID-002, US-ID-003, US-ID-004, US-ID-005, US-ID-006, US-ID-007, US-ID-008, US-UM-002, US-PM-002, US-SM-010 |

## Description

Make the Consent Manager an **OAuth2 resource server**. It issues no tokens, stores no credentials,
and holds no sessions. Every authenticated request carries a bearer token issued by an external,
explicitly trusted OpenID Connect provider.

This ticket replaces the entire pre-revision authentication design — four bespoke mechanisms,
password hashing, session storage and a half-built OAuth authorization server — with one mechanism
and a documented token contract.

This is the critical path. Nothing downstream can be built or tested without it.

## Technical Details

### 1. IDP Registry (Static Configuration)

Trusted providers are declared in configuration. Changing the trust list is a redeploy. There is no
registry table and no runtime administration API — a trust list is security-critical, changes
rarely, and a runtime API would need a bootstrap admin that could not itself be authenticated.

```yaml
consent-manager:
  identity-providers:
    - issuer: "https://idp.example.org/realms/dataspace"
      discovery-url: "https://idp.example.org/realms/dataspace/.well-known/openid-configuration"
      audience: "consent-manager"
      jwks-cache-ttl: 1h
      clock-skew: 30s
      claims:
        user-identifier: "sub"
        participant-identifier: "participant_id"
        roles: "realm_access.roles"
      role-mapping:
        USER: "dataspace-user"
        PARTICIPANT: "dataspace-participant"
        CATALOG: "dataspace-catalog"
```

**Startup behaviour:**
1. Bind the list to `@ConfigurationProperties`, validated with Bean Validation — a malformed or
   empty registry is a startup failure, not a runtime surprise.
2. For each entry, fetch the `discovery-url` and extract `issuer` and `jwks_uri`.
3. Reject the configuration if the discovered `issuer` does not exactly match the configured one.
4. Prime the JWKS cache.

Discovery failure at startup must **not** prevent the service from starting — an IDP outage would
otherwise take the whole service down. Log at ERROR, mark the readiness probe `DOWN`, and retry with
backoff. Liveness stays `UP`.

### 2. Token Validation Pipeline

For every request carrying `Authorization: Bearer <token>`:

1. Parse the JWS header and claims without trusting them.
2. Read `iss`. If it does not match a registered issuer → **401**, with a message that does not echo
   the issuer back (US-ID-008).
3. Resolve that issuer's JWKS from cache; select the key by `kid`. On cache miss or unknown `kid`,
   refetch once, then fail.
4. Verify the signature. Reject `alg: none` and any symmetric algorithm outright.
5. Validate `exp`, `iat`, and `nbf` if present, with the configured clock skew.
6. Validate that `aud` contains the configured audience for that issuer.
7. Extract the roles claim at the configured path and map to `USER`, `PARTICIPANT`, `CATALOG`.
   A token with no mapped role authenticates but authorizes nothing.
8. Extract the identifier claim at the configured path. A `USER`-role token without a resolvable
   user identifier → **401**. A `PARTICIPANT`-role token without a participant identifier → **401**.

Implemented with `micronaut-security-jwt` remote JWKS validators, one per registered issuer, plus a
custom `TokenValidator` for the claim and role mapping.

### 3. Token Contract (US-ID-003)

This is a **deliverable, not an implementation note**. Integrators configure their IDP against it.

It is authored directly in the API specification, in `api/components/security.yaml`, as
`securitySchemes` plus a description enumerating every claim below (TICKET-015). Because the
specification is written first and the code generated from it, the token contract is a design
artefact reviewed before the identity layer is built — not a description produced afterwards from
whatever the implementation turned out to require.

#### User Access Token

| Claim | Required | Purpose |
|-------|----------|---------|
| `iss` | yes | Must exactly match a registered issuer. Selects JWKS and claim mapping. |
| `aud` | yes | Must contain the configured audience for that issuer. |
| `exp`, `iat` | yes | Lifetime validation with configurable clock skew. |
| `sub` | yes | Subject at the IDP. |
| *user-identifier claim* | yes | The globally unique user identifier. Claim name configurable per IDP; defaults to `sub`. May be an email, a DID, a URI, or a hash derived from an EUDI Wallet PID — treated as opaque. |
| *roles claim* | yes | Must contain the role mapped to `USER`. |
| `email` | no | Stored for optional search and notification. Never used as a key. |
| `email_verified` | no | Recorded. Does not gate consent. |
| `name`, `given_name`, `family_name` | no | Display only. |
| `nbf`, `jti` | no | Validated if present. |

#### Participant Access Token

| Claim | Required | Purpose |
|-------|----------|---------|
| `iss`, `aud`, `exp`, `iat`, `sub` | yes | As above. |
| *participant-identifier claim* | yes | The data space global participant ID — DID, URI, or IDP-issued string. |
| *roles claim* | yes | Must contain the role mapped to `PARTICIPANT`. |
| `azp` / `client_id` | no | Recorded for audit. |

#### Catalog Token

As the participant token, but bearing the role mapped to `CATALOG`. This replaces the
`x-catalog-consent-key` shared secret entirely.

### 4. Authorization Model (US-SM-010)

Every endpoint declares its required role with `@Secured`. There is no implicit access.

| Role | Granted to | Scope |
|------|-----------|-------|
| `USER` | A data subject | Their own consents and profile only |
| `PARTICIPANT` | A registered organization | Their own participant record; consents and user links where they are provider or consumer |
| `CATALOG` | The data space catalog service | User lookup and consent resume |

**Resource-level checks are mandatory on top of the role check.** A `PARTICIPANT` token proves only
*which* organization is calling; the handler must still verify that the organization is the provider
or consumer of the consent it is touching. This is now the primary security boundary of the service
— the pre-revision design put the Consent Manager in the data path, where it could refuse to
forward; it now only records, so an authorization mistake is a direct cross-participant data leak.

Distinguish clearly:
- No token, invalid token, unknown issuer → **401**
- Valid token, wrong role or wrong organization → **403**

The role model is deliberately coarse. Attribute-based rules (per-purpose, per-data-category) can be
layered on later without changing the token contract.

### 5. Authenticated Principal Resolution

A request filter resolves the validated token into a typed principal available to handlers:

- `UserPrincipal` — global identifier, optional email and name claims, resolved `User` entity
- `ParticipantPrincipal` — global identifier, resolved `Participant` entity
- `CatalogPrincipal` — issuer and subject only

Handlers **never** read an identifier from the request body or a path parameter to decide who the
caller is. Identity comes from the token, always.

### 6. Just-in-Time User Provisioning (US-ID-005)

When a `USER`-role token presents an identifier with no matching `users` row, create one inside the
request, from claims only: `identifier` from the configured claim, plus `email`, `first_name`,
`last_name` if present. Provisioning is idempotent and safe under concurrency — rely on the unique
constraint on `users.identifier` and treat a duplicate-key violation as "already exists, re-read".

On each subsequent authenticated request, refresh the display claims if they changed. These fields
are non-authoritative copies; the IDP remains the source of truth.

JIT provisioning applies to **users only**. Participants are created explicitly (TICKET-005),
because a participant record carries a legal name and other metadata that no token supplies.

### 7. Key Rotation (US-ID-006)

JWKS responses are cached per issuer for `jwks-cache-ttl`. On encountering an unknown `kid`, refetch
once immediately, rate-limited to avoid a refetch storm from forged tokens. Rotation must never
require a restart. Cover this with a test that swaps the container IDP's signing key mid-test.

### Files to Create

```
src/main/java/eu/prometheusx/consentmanager/security/
  IdentityProviderConfiguration.java   # @ConfigurationProperties, validated
  IdentityProviderRegistry.java        # discovery, JWKS caching, per-issuer validators
  ConsentManagerTokenValidator.java    # signature, claims, audience, issuer
  ClaimMapper.java                     # configurable claim paths -> identifiers and roles
  PrincipalResolutionFilter.java       # typed principal on the request
  UserPrincipal.java
  ParticipantPrincipal.java
  CatalogPrincipal.java
  Role.java                            # USER, PARTICIPANT, CATALOG
src/main/java/eu/prometheusx/consentmanager/service/
  UserProvisioningService.java         # JIT provisioning from claims
```

## Acceptance Criteria

1. [ ] Configuration binding fails fast on a malformed or empty IDP registry
2. [ ] OIDC discovery resolves `jwks_uri` at startup and the service refuses an issuer mismatch
3. [ ] Discovery failure marks readiness `DOWN` without failing liveness or preventing startup
4. [ ] A token from an unregistered issuer is rejected with 401 and no issuer echoed in the response
5. [ ] A token with a bad signature, expired `exp`, or wrong `aud` is rejected with 401
6. [ ] `alg: none` and symmetric algorithms are rejected regardless of the key material
7. [ ] Claim paths are configurable per IDP, including nested paths such as `realm_access.roles`
8. [ ] Roles map to `USER`, `PARTICIPANT`, `CATALOG` per the configured mapping
9. [ ] A valid token with no mapped role receives 403, not 401, on a secured endpoint
10. [ ] A `USER` token for an unknown identifier provisions a user record from claims
11. [ ] Concurrent first-time requests for the same identifier create exactly one user row
12. [ ] Display claims are refreshed on subsequent requests when they change
13. [ ] Signing-key rotation at the IDP is picked up without a restart
14. [ ] A repeated unknown `kid` does not cause unbounded JWKS refetching
15. [ ] Handlers can obtain a typed principal and never read identity from the body or path
16. [ ] Parameterized tests cover the full matrix of (role × endpoint × expected status)
17. [ ] Integration tests run against a Testcontainers Keycloak instance with real tokens
18. [ ] The token contract is authored in `api/components/security.yaml` and published in the API specification
19. [ ] Every operation's `security` requirement matches the `@Secured` annotation on its implementing controller

## Replaced Mechanisms

Everything below existed in the pre-revision plan and is deliberately gone:

| Removed | Replaced by |
|---------|-------------|
| `verifyParticipantJWT`, including the `serviceKey` / `clientSecret` path | IDP token with `PARTICIPANT` role |
| `verifyUserJWT` — session, `x-user-key`, and OAuth paths | IDP token with `USER` role |
| `verifyUserKey` (`x-user-key` header) | Nothing. A participant can no longer act as a user. |
| `consentKeyCheck` (`x-catalog-consent-key` shared secret) | IDP token with `CATALOG` role |
| `?participant=<token>` query-parameter authentication | `Authorization: Bearer` only |
| `express-session` + MongoDB session store | Nothing — stateless |
| bcrypt password hashing, `SALT_ROUNDS` | Nothing — no credentials stored |
| OAuth authorize/token endpoints (was TICKET-016) | The external IDP is the authorization server |
| `setUserIdForParticipant` middleware | Users are addressed by global identifier; no resolution step |
| `objectIdFormatCheck` middleware | Path parameters are typed `UUID` or opaque identifier strings |
| `generateAccessToken` / `generateNewAccessToken` duplication (original bug 13.4) | The service issues no tokens |
| `req.session.user.oath` typo (original bug 17.2) | No session |
| ObjectId reference-equality comparison (original bug 17.4) | No ObjectIds |
