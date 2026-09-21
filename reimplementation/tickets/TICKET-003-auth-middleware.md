# TICKET-003: Authentication Middleware

## Metadata

| Field | Value |
|-------|-------|
| **Ticket ID** | TICKET-003 |
| **Title** | Authentication Middleware |
| **Priority** | Critical |
| **Complexity** | L (Large) |
| **Dependencies** | TICKET-001, TICKET-002 |
| **Related User Stories** | US-PM-002, US-PM-008, US-SM-012 |

## Description

Implement all four authentication mechanisms used by the Consent Manager, plus session management with MongoDB-backed store. These middleware functions protect API endpoints and establish the identity of the caller (user, participant, or catalog service).

## Technical Details

### Authentication Mechanism 1: Participant JWT (`verifyParticipantJWT`)

**File:** `src/middleware/auth.ts`

**Purpose:** Authenticate requests from registered participants (data providers and consumers).

**Flow:**
1. Extract JWT from `Authorization: Bearer <token>` header OR `?participant=<token>` query parameter
2. Decode without verification first to inspect the payload
3. **Standard path:** If payload contains a `sub` claim:
   - Verify the token against `JWT_SECRET_KEY`
   - Extract `sub` as the participant ID
   - Look up `Participant.findById(sub)`
   - Attach participant to `req.participant`
4. **DSC path:** If payload contains a `serviceKey` claim:
   - Look up `Participant.findOne({ clientID: serviceKey })`
   - Verify the token against that participant's `clientSecret`
   - Attach participant to `req.participant`
5. **Session shortcut:** If `req.session.participant` is already set, skip token verification and use the cached participant
6. On failure: return `401 Unauthorized`

### Authentication Mechanism 2: User JWT (`verifyUserJWT`)

**File:** `src/middleware/auth.ts`

**Purpose:** Authenticate end users via multiple strategies (session, user key header, OAuth token).

**Flow (try strategies in order, first success wins):**
1. **Session shortcut:** If `req.session.user` exists, attach to `req.user` and return
2. **User Key header:** If `x-user-key` header is present:
   - Look up `UserIdentifier.findById(x-user-key)`
   - Resolve the associated User via `UserIdentifier.user` or `UserIdentifier.email`
   - Auto-link the UserIdentifier to the User if not already linked
   - Attach user to `req.user`
3. **OAuth Bearer token:** If `Authorization: Bearer <token>` header is present:
   - Verify against `OAUTH_SECRET_KEY`
   - Extract user ID from token payload
   - Look up `User.findById(userId)`
   - Attach user to `req.user`
4. If none succeed: return `401 Unauthorized`

### Authentication Mechanism 3: User Key (`verifyUserKey`)

**File:** `src/middleware/auth.ts`

**Purpose:** Lightweight user identification for consent operations — resolves to a UserIdentifier only (not the full User).

**Flow:**
1. Read the `x-user-key` header value
2. Look up `UserIdentifier.findById(x-user-key)`
3. If found, attach to `req.userIdentifier`
4. If not found: return `401 Unauthorized`

### Authentication Mechanism 4: Catalog Consent Key (`consentKeyCheck`)

**File:** `src/middleware/consentKeyCheck.ts`

**Purpose:** Authenticate inter-service calls from the Catalog service.

**Flow:**
1. Read the `x-catalog-consent-key` header value
2. Compare against `X_CATALOG_CONSENT_KEY` environment variable
3. If match: proceed to next middleware
4. If no match or missing: return `401 Unauthorized`

### Session Management

**File:** `src/middleware/session.ts`

**Configuration:**
- Store: `connect-mongo` with the active MongoDB connection
- Cookie max age: 24 hours
- `secure: true` in production
- `sameSite: "strict"` in production
- Session secret from `SESSION_SECRET` environment variable

### Participant Middleware: `setUserIdForParticipant`

**File:** `src/middleware/participantsMiddleware.ts`

**Purpose:** Resolve a UserIdentifier to its linked User and attach the User ID for participant-scoped operations.

**Flow:**
1. Extract `userIdentifier` ID from request (body or params)
2. Look up `UserIdentifier.findById(id)`
3. Resolve the User via the identifier's `user` field or email lookup
4. Attach `req.userId` for downstream controllers

### ObjectId Format Validation

**File:** `src/middleware/objectIdFormatCheck.ts`

**Purpose:** Validate that route parameters containing MongoDB ObjectIds are in the correct 24-character hex format before reaching controllers.

### Files to Create

```
src/middleware/
  auth.ts                    # verifyParticipantJWT, verifyUserJWT, verifyUserKey
  consentKeyCheck.ts         # consentKeyCheck
  session.ts                 # Express session configuration
  participantsMiddleware.ts  # setUserIdForParticipant
  objectIdFormatCheck.ts     # ObjectId format validation
src/libs/
  jwt/
    index.ts                 # JWT utility functions (generate, verify, refresh)
```

### JWT Utility Functions (`src/libs/jwt/`)

| Function | Purpose |
|----------|---------|
| `generateAccessToken(payload, secret, expiresIn)` | Create a signed JWT |
| `generateRefreshToken(payload, secret, expiresIn)` | Create a refresh token |
| `verifyToken(token, secret)` | Verify and decode a JWT |

**Note:** The original codebase has `generateNewAccessToken` as an exact duplicate of `generateAccessToken` (bug 13.4). Do not replicate this — implement a single function.

## Acceptance Criteria

1. [ ] `verifyParticipantJWT` authenticates via Bearer token with `sub` claim (standard path)
2. [ ] `verifyParticipantJWT` authenticates via Bearer token with `serviceKey` claim (DSC path)
3. [ ] `verifyParticipantJWT` accepts `?participant=<token>` query parameter
4. [ ] `verifyParticipantJWT` uses session cache when available
5. [ ] `verifyUserJWT` tries session, then `x-user-key` header, then OAuth Bearer in order
6. [ ] `verifyUserKey` resolves `x-user-key` header to a UserIdentifier
7. [ ] `consentKeyCheck` validates the `x-catalog-consent-key` header against the env var
8. [ ] Session middleware is configured with MongoDB store and 24h cookie
9. [ ] `setUserIdForParticipant` resolves UserIdentifier to User
10. [ ] `objectIdFormatCheck` rejects invalid ObjectId formats with 400
11. [ ] All middleware returns proper HTTP status codes on failure (401, 400)
12. [ ] Unit tests cover all authentication paths, including edge cases
13. [ ] JWT utility exports a single token generation function (no duplicates)

## Known Issues to Address

1. **`generateNewAccessToken` is an exact duplicate** (original bug 13.4): Implement a single `generateAccessToken` function. Do not duplicate.
2. **Session key typo** (original bug 17.2): The original stores tokens under `req.session.user.oath` (misspelled). Use `req.session.user.oauth` correctly.
3. **ObjectId comparison uses reference equality** (original bug 17.4): When comparing ObjectIds in arrays, use `.equals()` or convert to strings, not `Array.includes()` with `===`.
