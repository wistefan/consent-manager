# TICKET-004: User Management

## Metadata

| Field | Value |
|-------|-------|
| **Ticket ID** | TICKET-004 |
| **Title** | User Management |
| **Priority** | Critical |
| **Complexity** | M (Medium) |
| **Dependencies** | TICKET-002, TICKET-003 |
| **Related User Stories** | US-UM-001, US-UM-002, US-UM-003, US-UM-004, US-UM-005, US-UM-006 |

## Description

Implement user signup, login, logout, profile retrieval, and search endpoints. This covers the core user account lifecycle without UserIdentifier management (which is handled in TICKET-006).

## Technical Details

### Endpoints

#### 1. User Signup — `POST /users/signup`

**Auth:** None (public)

**Request Body:**
```json
{
  "firstName": "string (required)",
  "lastName": "string (required)",
  "email": "string (required)",
  "password": "string (required)"
}
```

**Flow:**
1. Validate request body
2. Check for existing user with same email
3. Create new User document (password auto-hashed by pre-save hook)
4. Generate OAuth access token and refresh token
5. Store tokens in `user.oauth`
6. Generate JSON-LD self-description
7. Create session with `req.session.user`
8. Return user data (excluding sensitive fields)

**Response:** `201 Created` with user object, access token, and refresh token

#### 2. User Login — `POST /users/login`

**Auth:** None (public)

**Request Body:**
```json
{
  "email": "string (required)",
  "password": "string (required)"
}
```

**Flow:**
1. Find user by email
2. Validate password using `user.validatePassword()`
3. Generate new access token and refresh token
4. Update `user.oauth` with new tokens
5. Create session
6. Return user data with tokens

**Response:** `200 OK` with user object and tokens

#### 3. User Logout — `DELETE /users/logout`

**Auth:** `verifyUserJWT`

**Flow:**
1. Destroy the express session (`req.session.destroy()`)
2. Return success

**Response:** `200 OK`

#### 4. View My Profile — `GET /users/me`

**Auth:** `verifyUserJWT`

**Flow:**
1. Read authenticated user from `req.user`
2. Generate fresh access token and refresh token
3. Return user data with new tokens

**Response:** `200 OK` with user object and refreshed tokens

#### 5. Look Up User by ID — `GET /users/:userId`

**Auth:** None (public)

**Flow:**
1. Validate `:userId` is a valid ObjectId
2. Find user by ID, excluding password and sensitive fields
3. Return user data

**Response:** `200 OK` with user object (public fields only)

#### 6. Search User by Email — `POST /users/search`

**Auth:** `consentKeyCheck` (catalog service only)

**Request Body:**
```json
{
  "email": "string (required)"
}
```

**Flow:**
1. Validate the catalog consent key header
2. Find user by email
3. Return user data if found

**Response:** `200 OK` with user object or `404 Not Found`

### Files to Create/Modify

```
src/controllers/usersController.ts   # Controller functions
src/routes/users.ts                  # Route definitions
src/routes/index.ts                  # Mount users router
```

### Response Field Selection

Define a constant for public user fields to exclude sensitive data:

```typescript
const USER_PUBLIC_FIELDS = "firstName lastName email identifiers jsonld createdAt updatedAt";
```

Never return `password`, `oauth.refreshToken`, or internal fields in responses.

## Acceptance Criteria

1. [ ] `POST /users/signup` creates a user with hashed password and returns tokens
2. [ ] `POST /users/signup` rejects duplicate emails with `409 Conflict`
3. [ ] `POST /users/login` authenticates with email/password and returns tokens
4. [ ] `POST /users/login` returns `401 Unauthorized` for invalid credentials
5. [ ] `DELETE /users/logout` destroys the session
6. [ ] `GET /users/me` returns the authenticated user with refreshed tokens
7. [ ] `GET /users/:userId` returns public user data without password
8. [ ] `POST /users/search` requires catalog consent key authentication
9. [ ] Signup response never includes password hash or OAuth refresh token
10. [ ] All endpoints return consistent error formats via global error handler
11. [ ] Integration tests cover signup, login, logout, profile, and search flows

## Known Issues to Address

1. **Signup leaks password hash** (original bug 17.1): Use field selection to exclude `password`, `oauth`, and other sensitive fields from all responses.
2. **Session key typo** (original bug 17.2): Store session data under `req.session.user.oauth` (not `oath`).
3. **JSON-LD never regenerated** (original bug 17.7): Consider regenerating `jsonld` on profile updates, or document this as a known limitation.
4. **Dead route** (original bug 17.8): Do not implement `GET /users/:userId/:consentId` — it has no handler in the original. Only implement documented, functional routes.
