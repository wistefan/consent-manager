# User Management

This document provides comprehensive feature documentation for the user management module. It covers user authentication (signup, login, logout, profile), UserIdentifier registration and lifecycle, cross-participant user federation, and user lookup/search endpoints.

> **Source files:** `src/controllers/usersController.ts`, `src/routes/users.ts`, `src/models/User/User.model.ts`, `src/models/UserIdentifier/UserIdentifier.model.ts`, `src/middleware/auth.ts`, `src/middleware/participantsMiddleware.ts`, `src/utils/UserIdentifierMatchingProcessor.ts`, `src/libs/user-reconcilation/index.ts`, `src/libs/jsonld/selfDescriptions.ts`

---

## Table of Contents

1. [User Signup](#1-user-signup)
2. [User Login](#2-user-login)
3. [User Logout](#3-user-logout)
4. [User Profile (Me)](#4-user-profile-me)
5. [User Lookup](#5-user-lookup)
6. [User Search by Email](#6-user-search-by-email)
7. [UserIdentifier Registration (Single)](#7-useridentifier-registration-single)
8. [UserIdentifier Registration (Bulk)](#8-useridentifier-registration-bulk)
9. [UserIdentifier Search](#9-useridentifier-search)
10. [UserIdentifier Injection](#10-useridentifier-injection)
11. [UserIdentifier Update](#11-useridentifier-update)
12. [UserIdentifier Delete](#12-useridentifier-delete)
13. [Cross-Participant User Federation](#13-cross-participant-user-federation)
14. [Participant Middleware: User Resolution](#14-participant-middleware-user-resolution)
15. [JSON-LD Self-Description](#15-json-ld-self-description)
16. [Endpoint Reference](#16-endpoint-reference)
17. [Known Bugs](#17-known-bugs)

---

## 1. User Signup

**Endpoint:** `POST /users/signup` `[None]`

**Request body:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `firstName` | string | yes | User's first name |
| `lastName` | string | yes | User's last name |
| `email` | string | yes | User's email address (must be unique) |
| `password` | string | yes | Plaintext password (hashed via bcrypt pre-save hook) |

**Logic flow:**

1. Check for existing user with the same email via `User.findOne({ email })`.
2. If duplicate found, return `409`.
3. Create a new `User` document with the provided fields.
4. Generate an OAuth access token (JWT, 1-hour expiry, signed with `OAUTH_SECRET_KEY`) with scopes `[userRead, userWrite]`.
5. Generate a refresh token (random 32-byte hex string).
6. Set `user.oauth = { scopes: [userRead, userWrite], refreshToken }`.
7. Generate a JSON-LD self-description string (schema.org Person) and set `user.jsonld`.
8. Save the user -- the pre-save hook hashes the password with bcrypt (salt rounds from `SALT_ROUNDS` env var, default 10).
9. Return the user, access token, and refresh token.

**Response:** `200`
```json
{
  "user": { /* full User document */ },
  "accessToken": "<JWT>",
  "refreshToken": "<hex string>"
}
```

**Errors:**
- `409` -- `{ message: "User with this email address already exists" }`

> **⚠ Bug:** The response returns the full `user` object without field selection, leaking the hashed password. See [Known Bugs](#171-signup-leaks-password-hash).

---

## 2. User Login

**Endpoint:** `POST /users/login` `[None]`

**Request body:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `email` | string | yes | User's email address |
| `password` | string | yes | Plaintext password |

**Logic flow:**

1. Find user by email via `User.findOne({ email })`.
2. Validate password using `user.validatePassword(password)` (bcrypt compare).
3. Generate new access token (JWT, 1h) and refresh token (random 32 bytes hex).
4. Store tokens in `req.session.user` for session-based auth on subsequent requests.
5. Update `user.oauth.refreshToken` and save.
6. Re-query user with `USER_SELECTION` (`-password -oauth`) for sanitized output.

**Response:** `200`
```json
{
  "user": { /* sanitized User (no password, no oauth) */ },
  "accessToken": "<JWT>",
  "refreshToken": "<hex string>"
}
```

**Errors:**
- `404` -- `{ message: "User not found" }`
- `400` -- `{ message: "invalid credentials" }`

> **⚠ Bug:** The session object key is misspelled as `oath` instead of `oauth` (`req.session.user.oath`). See [Known Bugs](#172-session-key-typo).

---

## 3. User Logout

**Endpoint:** `DELETE /users/logout` `[None]`

**Logic flow:**

1. If `req.session` does not exist, return `200 "OK"` (plain text).
2. Call `req.session.destroy()`.
3. On completion, return JSON response.

**Response:** `200`
```json
{
  "data": {},
  "message": "successful logout"
}
```

> **Note:** The logout handler is defined inline in the routes file (`src/routes/users.ts`), not in the controller.

---

## 4. User Profile (Me)

**Endpoint:** `GET /users/me` `[UserJWT]`

**Logic flow:**

1. Read `req.user.id` from the `verifyUserJWT` middleware.
2. Find user by ID, excluding the password field.
3. Generate new access token and refresh token.
4. Store in session and update `user.oauth.refreshToken`.
5. Re-query with `USER_SELECTION` for sanitized output.

**Response:** `200`
```json
{
  "user": { /* sanitized User */ },
  "accessToken": "<JWT>",
  "refreshToken": "<hex string>"
}
```

**Errors:**
- `404` -- `{ message: "User not found" }`

> **Note:** This endpoint effectively refreshes the user's tokens on every call. The JSDoc on the controller function incorrectly says "Logs in a user".

---

## 5. User Lookup

**Endpoint:** `GET /users/:userId/` `[None]`

**Parameters:**
- `userId` -- MongoDB ObjectId of the User

**Logic flow:**

1. Find user by ID with `USER_SELECTION` (excludes password and oauth fields).
2. Return the sanitized user document.

**Response:** `200` -- Sanitized User object

**Errors:**
- `404` -- `{ message: "User not found" }`

> **Note:** This is a public endpoint with no authentication. It exposes user data (name, email, identifiers, jsonld) to any caller.

---

## 6. User Search by Email

**Endpoint:** `POST /users/search` `[CatalogKey]`

**Request body:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `email` | string | yes | Email address to search for |

**Logic flow:**

1. Validate the `x-catalog-consent-key` header (via `consentKeyCheck` middleware).
2. Find user by email via `User.findOne({ email })`, excluding `-oauth -password`.
3. Return the user document or 404.

**Response:** `200` -- User object (without oauth and password)

**Errors:**
- `401` -- Invalid or missing catalog consent key
- `404` -- `{ message: "User not found" }`

---

## 7. UserIdentifier Registration (Single)

**Endpoint:** `POST /users/register` `[ParticipantJWT]`

**Request body:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `email` | string | conditional | Required if `identifier` is absent |
| `identifier` | string | conditional | Required if `email` is absent (alternative DID-style identifier) |
| `url` | string | no | Dynamic URL for the user at the participant |

**Logic flow:**

1. Validate that at least one of `email` or `identifier` is present.
2. Check for existing UserIdentifier with the same `attachedParticipant` + `email`.
3. If duplicate found, return `409`.
4. Create a new `UserIdentifier` with:
   - `attachedParticipant` = authenticated participant's ID (`req.userParticipant.id`)
   - `email`, `identifier`, `url` from the request body
5. Save the UserIdentifier.
6. Call `checkUserIdentifier(email, participantId, newId._id)` to trigger cross-participant user federation (see [Section 13](#13-cross-participant-user-federation)).
7. Return the new UserIdentifier document.

**Response:** `200` -- New UserIdentifier document

**Errors:**
- `400` -- `BadRequestError` if both `email` and `identifier` are missing
- `409` -- `{ error: "User already exists" }`

---

## 8. UserIdentifier Registration (Bulk)

**Endpoint:** `POST /users/registers` `[ParticipantJWT]`

**Request body:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `users` | array | yes | Array of user objects |
| `users[].email` | string | conditional | Required if `internalID` is absent |
| `users[].internalID` | string | conditional | Required if `email` is absent; mapped to `identifier` field |
| `users[].url` | string | no | Dynamic URL |

**Logic flow:**

1. Iterate over the `users` array.
2. For each user, check for an existing UserIdentifier with the same `attachedParticipant` + `email`.
3. Validate that at least one of `email` or `identifier` is present.
4. If the identifier does not already exist, create and save a new `UserIdentifier`.
5. Collect all newly created identifiers in the response array.
6. Return the array of new UserIdentifiers (skipping already-existing ones silently).

**Response:** `200` -- Array of new UserIdentifier documents

**Errors:**
- `400` -- `BadRequestError` if any user lacks both `email` and `identifier`

> **⚠ Bug:** The bulk endpoint does **not** call `checkUserIdentifier`, so cross-participant federation is skipped entirely for bulk-registered users. See [Known Bugs](#173-bulk-registration-skips-federation).

---

## 9. UserIdentifier Search

**Endpoint:** `POST /users/identifier/search` `[CatalogKey]`

**Request body:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `selfDescription` | string | yes | Self-description URL of the participant |
| `email` | string | yes | Email address to search for |

**Logic flow:**

1. Validate the `x-catalog-consent-key` header.
2. Find the Participant by `selfDescriptionURL` matching `selfDescription`.
3. If participant found, search for a UserIdentifier with that participant's ID and the given email.
4. If UserIdentifier found, search for Users with the same email and check if any User's `identifiers` array contains this UserIdentifier's ID.
5. Build and return a status object.

**Response:** `200` -- Always returns 200 regardless of whether entities are found.
```json
{
  "participantExists": true,
  "participantId": "<ObjectId>",
  "userIdentifierExists": true,
  "userIdentifier": "<ObjectId>",
  "userExists": true
}
```

> **Note:** This endpoint performs an existence check and never returns the full User or UserIdentifier documents. The `userExists` flag specifically checks whether the UserIdentifier is linked to a User (not just whether a User with that email exists).

---

## 10. UserIdentifier Injection

**Endpoint:** `POST /users/identifier` `[CatalogKey]`

**Request body:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `userId` | string | yes | MongoDB ObjectId of the target User |
| `userIdentifiers` | string[] | yes | Array of UserIdentifier ObjectIds to attach |

**Logic flow:**

1. Validate the `x-catalog-consent-key` header.
2. Find the User by `userId`.
3. For each identifier ID in the `userIdentifiers` array:
   - Verify the UserIdentifier document exists.
   - Skip if already present in `user.identifiers`.
   - Push the ObjectId into `user.identifiers`.
4. Save the user.
5. Re-query with sanitized fields (no password, no oauth).

**Response:** `200` -- Sanitized User document

**Errors:**
- `404` -- `{ message: "User not found" }` or `{ message: "User Identifier not found" }`

> **⚠ Bug:** The `Array.includes` check for duplicate ObjectIds may produce false negatives because Mongoose ObjectIds are objects, not primitives. See [Known Bugs](#174-objectid-comparison-in-inject).

---

## 11. UserIdentifier Update

**Endpoint:** `PUT /users/:id` `[ParticipantJWT]`

**Parameters:**
- `id` -- MongoDB ObjectId of the **UserIdentifier** (not the User)

**Request body:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `url` | string | no | Updated URL |
| `internalID` | string | no | Updated identifier (mapped to `identifier` field) |
| `email` | string | no | Updated email |

**Logic flow:**

1. Call `UserIdentifier.findByIdAndUpdate(id, { url, identifier: internalID, email })`.
2. Return the document.

**Response:** `200` -- UserIdentifier document

**Errors:**
- `404` -- `{ message: "User not found" }`

> **⚠ Bug:** The `findByIdAndUpdate` call does not use `{ new: true }`, so it returns the **pre-update** document. See [Known Bugs](#175-update-returns-stale-data).

---

## 12. UserIdentifier Delete

**Endpoint:** `DELETE /users/:id` `[ParticipantJWT]`

**Parameters:**
- `id` -- MongoDB ObjectId of the **UserIdentifier**

**Logic flow:**

1. Call `UserIdentifier.findByIdAndDelete(id)`.
2. Return the deleted document.

**Response:** `200` -- Deleted UserIdentifier document

**Errors:**
- `404` -- `{ message: "User not found" }`

> **⚠ Bug:** Deleting a UserIdentifier does **not** remove its reference from `User.identifiers`. Orphaned ObjectId references remain in the User document. See [Known Bugs](#176-delete-orphans-references).

---

## 13. Cross-Participant User Federation

The system provides cross-participant user identity linking via email matching. When a participant registers a user (UserIdentifier), the system checks whether the same email has been registered by a **different** participant and automatically links them to a shared User account.

### 13.1 `checkUserIdentifier` (`src/utils/UserIdentifierMatchingProcessor.ts`)

**Signature:** `checkUserIdentifier(email, participantId, userIdentifier, existingUser?)`

**Algorithm:**

1. **Find a cross-participant match:** Query `UserIdentifier.findOne({ attachedParticipant: { $ne: participantId }, email })` -- find a UserIdentifier from any **other** participant with the same email.
2. **If no match found:** Return `undefined` (no federation needed).
3. **If match found, locate the User:**
   - If `existingUser` parameter is provided, use it directly.
   - Otherwise, query `User.findOne({ $or: [{ identifiers: matchedIdentifier._id }, { email }] })`.
4. **If a User exists:**
   - Push both the matched identifier and the new identifier into `user.identifiers` (deduplicating with `Array.includes`).
   - Save the User.
5. **If no User exists:**
   - Create a new `User({ email, identifiers: [matchedId, newId] })` -- this user has **no password, firstName, or lastName**.
   - Save the User.
6. Return the User.

**Called from:**
- `registerUserIdentifier` (single registration, `src/controllers/usersController.ts`)
- `giveConsent` / `giveConsentUser` flows in `src/controllers/consentsController.ts` (during consent granting when a new UserIdentifier is created for the consumer side)

### 13.2 `TryFindMatchingUser` (`src/libs/user-reconcilation/index.ts`)

**Signature:** `TryFindMatchingUser(userIdentifier: IUserIdentifier)`

A simple utility that queries `User.findOne({ email: userIdentifier.email })`. Returns the matched User or `null`.

> **Note:** This function is defined but not imported by any user-facing controller. It is only referenced in the implementation plan documentation.

### 13.3 Implicit Federation in `verifyUserJWT` Middleware

The `x-user-key` authentication path in the `verifyUserJWT` middleware (`src/middleware/auth.ts`) also performs implicit federation:

1. Look up the UserIdentifier by the `x-user-key` header value.
2. Find a User whose `identifiers` array contains the UserIdentifier.
3. If no match by identifier, fall back to `User.findOne({ email: userIdentifier.email })`.
4. If found by email but the UserIdentifier is not yet in `user.identifiers`, **push it in and save**.

This means that authenticating via `x-user-key` can trigger user-identity linking as a side effect.

---

## 14. Participant Middleware: User Resolution

**Middleware:** `setUserIdForParticipant` (`src/middleware/participantsMiddleware.ts`)

**Purpose:** Resolves a UserIdentifier ObjectId (passed as a URL parameter) to the actual User ObjectId. Used by consent endpoints that accept a `userId` param representing a UserIdentifier.

**Logic:**

1. Read `req.params.userId` (which is actually a UserIdentifier ObjectId).
2. Validate that it is a valid MongoDB ObjectId format.
3. Query `User.findOne({ identifiers: userIdentifier })` to find the parent User.
4. Set `req.user = { id: user._id }`.
5. Throw `NotFoundError` if no User is found.

---

## 15. JSON-LD Self-Description

**Function:** `userToSelfDescription(user)` (`src/libs/jsonld/selfDescriptions.ts`)

Generates a schema.org Person JSON-LD string:

```json
{
  "@context": "http://schema.org",
  "@type": "Person",
  "name": "<firstName> <lastName>",
  "email": "<email>",
  "url": "<URL>:<PORT if dev>/v1/users/<user.id>"
}
```

**When called:** Only during `signup`. The `jsonld` field is never regenerated when a user's name or email changes.

> **⚠ Bug:** If the user's name or email is updated, the `jsonld` field becomes stale. See [Known Bugs](#177-json-ld-never-regenerated).

---

## 16. Endpoint Reference

| # | Method | Path | Auth | Controller | Description |
|---|--------|------|------|------------|-------------|
| 1 | `POST` | `/users/signup` | None | `signup` | Register a new user |
| 2 | `POST` | `/users/login` | None | `login` | Authenticate with email/password |
| 3 | `DELETE` | `/users/logout` | None | *(inline)* | Destroy session |
| 4 | `GET` | `/users/me` | UserJWT | `me` | Get current user + refresh tokens |
| 5 | `POST` | `/users/identifier/search` | CatalogKey | `getUserIdentifierByEmail` | Check existence of participant/identifier/user |
| 6 | `POST` | `/users/search` | CatalogKey | `getUserByEmail` | Find user by email |
| 7 | `POST` | `/users/identifier` | CatalogKey | `injectUserIdentifier` | Attach UserIdentifier IDs to a User |
| 8 | `POST` | `/users/register` | ParticipantJWT | `registerUserIdentifier` | Register single UserIdentifier |
| 9 | `POST` | `/users/registers` | ParticipantJWT | `registerUserIdentifiers` | Register bulk UserIdentifiers |
| 10 | `PUT` | `/users/:id` | ParticipantJWT | `updateUserById` | Update a UserIdentifier |
| 11 | `DELETE` | `/users/:id` | ParticipantJWT | `deleteUserById` | Delete a UserIdentifier |
| 12 | `GET` | `/users/:userId/` | None | `getUserById` | Public user lookup |
| 13 | `GET` | `/users/:userId/:consentId` | None | *(none)* | **Dead route** -- registered with no handler |

---

## 17. Known Bugs

The following bugs exist in the original implementation and should be corrected in a reimplementation:

### 17.1 Signup Leaks Password Hash

**Location:** `src/controllers/usersController.ts` (`signup` function)

**Description:** The `signup` response returns the full `user` object without any field selection. This includes the bcrypt-hashed password, OAuth tokens, and refresh token. All other endpoints (`login`, `me`, `getUserById`) use `USER_SELECTION` (`-password -oauth`) to exclude sensitive fields.

**Recommendation:** Apply `USER_SELECTION` to the signup response, or re-query the user with field filtering before returning.

### 17.2 Session Key Typo

**Location:** `src/controllers/usersController.ts` (`login` and `me` functions)

**Description:** The session object stores tokens under `req.session.user.oath` (misspelled) instead of `req.session.user.oauth`. While this has no runtime impact (the session key is arbitrary), it is inconsistent with the `User.oauth` field naming.

### 17.3 Bulk Registration Skips Federation

**Location:** `src/controllers/usersController.ts` (`registerUserIdentifiers` function)

**Description:** The bulk registration endpoint (`POST /users/registers`) does **not** call `checkUserIdentifier` after creating each UserIdentifier. This means users registered in bulk bypass cross-participant federation entirely. The singular endpoint (`POST /users/register`) calls `checkUserIdentifier` correctly.

**Impact:** If participant A bulk-registers users that overlap with participant B's users by email, those users will not be automatically linked.

### 17.4 ObjectId Comparison in Inject

**Location:** `src/controllers/usersController.ts` (`injectUserIdentifier` function)

**Description:** The deduplication check `user.identifiers.includes(new mongoose.Types.ObjectId(userIdentifier))` may produce false negatives. Mongoose ObjectIds are objects, and `Array.includes` uses `===` reference equality, which may fail even when two ObjectIds represent the same value.

**Recommendation:** Use `.toString()` comparison or Mongoose's built-in comparison methods.

### 17.5 Update Returns Stale Data

**Location:** `src/controllers/usersController.ts` (`updateUserById` function)

**Description:** `UserIdentifier.findByIdAndUpdate(id, ...)` is called without `{ new: true }`, so the response contains the **pre-update** document rather than the updated version.

### 17.6 Delete Orphans References

**Location:** `src/controllers/usersController.ts` (`deleteUserById` function)

**Description:** When a UserIdentifier is deleted, its ObjectId reference in `User.identifiers` is not removed. This leaves orphaned references that could cause issues in queries that attempt to populate or look up the deleted identifier.

**Recommendation:** Also update `User.updateMany({ identifiers: id }, { $pull: { identifiers: id } })` when deleting a UserIdentifier.

### 17.7 JSON-LD Never Regenerated

**Location:** `src/controllers/usersController.ts` and `src/libs/jsonld/selfDescriptions.ts`

**Description:** The `user.jsonld` field is only set during `signup`. If the user's name or email changes, the self-description becomes stale. No update endpoint regenerates it.

### 17.8 Dead Route

**Location:** `src/routes/users.ts` (line 52)

**Description:** The route `GET /users/:userId/:consentId` is registered but has **no handler function**. Any request to this path will hang or timeout because `next()` is never called.

### 17.9 No Unique Index on User Email

**Location:** `src/models/User/User.model.ts`

**Description:** The `email` field has no unique index at the Mongoose schema level. Uniqueness is enforced only at the application level in the `signup` function. A race condition could allow duplicate emails if two concurrent signup requests arrive for the same email.

**Recommendation:** Add a unique index on `User.email` at the schema level.

### 17.10 `user` Field on UserIdentifier Schema Is Unused

**Location:** `src/models/UserIdentifier/UserIdentifier.model.ts`

**Description:** The `user` field (ObjectId ref to `User`) exists in the UserIdentifier schema but is never set by any controller. The parent-child relationship is maintained only via `User.identifiers[]` (one-directional). This creates a misleading schema field.
