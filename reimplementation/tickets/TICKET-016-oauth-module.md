# TICKET-016: OAuth Module

## Metadata

| Field | Value |
|-------|-------|
| **Ticket ID** | TICKET-016 |
| **Title** | OAuth Module |
| **Priority** | Low |
| **Complexity** | M (Medium) |
| **Dependencies** | TICKET-003 |
| **Related User Stories** | US-SM-010 |

## Description

Implement the OAuth authorization and token endpoints. In the original codebase, this module exists as dead code (routes are defined but never mounted in the route loader). The reimplementation should either implement it properly or explicitly defer it as a documented future feature.

## Technical Details

### Current State (Original Codebase)

The original has:
- `src/routes/oauth.ts` — Defines `GET /authorize` and `POST /token` routes
- `src/middleware/oauth.ts` — Defines `validateAccessToken` middleware
- Neither is mounted in `src/routes/index.ts`

### Recommended Approach: Implement as Optional Module

#### OAuth Authorization Endpoint — `GET /oauth/authorize`

**Auth:** `verifyUserJWT` (user must be logged in)

**Query Parameters:**
- `response_type` — Must be `"code"`
- `client_id` — Participant's `clientID`
- `redirect_uri` — Callback URL for the participant
- `scope` — Requested scopes (space-separated)
- `state` — Anti-CSRF token

**Flow:**
1. Validate required parameters
2. Look up participant by `client_id`
3. Verify `redirect_uri` is registered for this participant
4. Display consent page (EJS template) asking user to authorize
5. On approval, generate authorization code
6. Redirect to `redirect_uri` with `code` and `state`

#### OAuth Token Endpoint — `POST /oauth/token`

**Auth:** None (client authenticates via `client_id` + `client_secret` in body or Basic auth)

**Request Body:**
```json
{
  "grant_type": "authorization_code",
  "code": "string (required)",
  "redirect_uri": "string (required)",
  "client_id": "string (required)",
  "client_secret": "string (required)"
}
```

**Flow:**
1. Validate the authorization code
2. Verify `client_id` and `client_secret`
3. Verify `redirect_uri` matches the original request
4. Generate access token and refresh token
5. Return tokens

**Response:**
```json
{
  "access_token": "string",
  "token_type": "Bearer",
  "expires_in": 3600,
  "refresh_token": "string",
  "scope": "string"
}
```

#### OAuth Scopes

| Scope | Description |
|-------|-------------|
| `userRead` | Read user profile data |
| `userWrite` | Modify user profile data |
| `organizationRead` | Read participant/organization data |
| `organizationWrite` | Modify participant/organization data |

#### Token Validation Middleware — `validateAccessToken`

Middleware that validates an OAuth access token and attaches the user/scopes to the request. Can be used as an alternative to `verifyUserJWT` for OAuth-authenticated endpoints.

### Files to Create

```
src/routes/oauth.ts                # OAuth routes
src/controllers/oauthController.ts # OAuth logic
src/middleware/oauth.ts            # Token validation middleware
src/views/consent.ejs              # Authorization consent page (EJS template)
```

### Mounting

Mount the OAuth routes in `src/routes/index.ts` (unlike the original where they are dead code):

```typescript
router.use('/oauth', oauthRouter);
```

Make the mounting conditional on an environment flag if desired:
```typescript
if (process.env.ENABLE_OAUTH === 'true') {
  router.use('/oauth', oauthRouter);
}
```

## Acceptance Criteria

1. [ ] `GET /oauth/authorize` displays an authorization consent page
2. [ ] Authorization flow generates an authorization code and redirects
3. [ ] `POST /oauth/token` exchanges authorization code for access + refresh tokens
4. [ ] `validateAccessToken` middleware validates OAuth tokens
5. [ ] OAuth routes are properly mounted (not dead code)
6. [ ] OAuth can be disabled via environment variable if not ready
7. [ ] Scopes are enforced on protected endpoints
8. [ ] Integration tests cover the full OAuth authorization code flow

## Known Issues to Address

1. **Dead code in original** (routes never mounted): Ensure OAuth routes are mounted or explicitly gated behind a feature flag.
2. **`generateNewAccessToken` duplicate** (original bug 13.4): Use a single token generation function (from TICKET-003).
3. **No authorization code storage**: The original has no mechanism to store/validate authorization codes. Implement with MongoDB or in-memory store with TTL.
4. **No redirect_uri validation**: Implement redirect URI registration and validation for security.
