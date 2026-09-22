# TICKET-006: UserIdentifier Management

## Metadata

| Field | Value |
|-------|-------|
| **Ticket ID** | TICKET-006 |
| **Title** | UserIdentifier Management |
| **Priority** | Critical |
| **Complexity** | L (Large) |
| **Dependencies** | TICKET-002, TICKET-003, TICKET-004, TICKET-005 |
| **Related User Stories** | US-UM-007, US-UM-008, US-UM-009, US-UM-010, US-UM-011, US-UM-012, US-UM-013 |

## Description

Implement UserIdentifier CRUD operations, cross-participant user federation, and bulk registration. UserIdentifiers are the mechanism by which a single User is recognized across multiple participants — each participant assigns its own identifier to a user, and the Consent Manager links them to the same User account via email matching.

## Technical Details

### Endpoints

#### 1. Register Single UserIdentifier — `POST /users/register`

**Auth:** `verifyParticipantJWT`

**Request Body:**
```json
{
  "email": "string (required)",
  "identifier": "string (optional)",
  "url": "string (optional — participant self-description URL)"
}
```

**Flow:**
1. Extract authenticated participant from `req.participant`
2. Create a new UserIdentifier document:
   - `attachedParticipant`: participant ID
   - `email`: from body
   - `identifier`: from body (optional)
   - `url`: from body (optional)
3. Run cross-participant federation via `checkUserIdentifier()`:
   - Look up existing Users with matching email
   - If a matching User is found, link the UserIdentifier to that User
   - Add the UserIdentifier's ID to the User's `identifiers` array
4. Return the created UserIdentifier

**Response:** `201 Created`

#### 2. Register UserIdentifiers in Bulk — `POST /users/registers`

**Auth:** `verifyParticipantJWT`

**Request Body:**
```json
{
  "userIdentifiers": [
    { "email": "string", "identifier": "string", "url": "string" },
    ...
  ]
}
```

**Flow:**
1. For each identifier in the array:
   a. Create UserIdentifier with `attachedParticipant` set to the authenticated participant
   b. Run `checkUserIdentifier()` for each (fixing original bug where bulk skips federation)
2. Return all created UserIdentifiers

**Response:** `201 Created` with array of UserIdentifiers

#### 3. Search for UserIdentifier — `POST /users/identifier/search`

**Auth:** `verifyParticipantJWT`

**Request Body:**
```json
{
  "participantId": "string (required — participant self-description URL, base64-encoded)",
  "userIdentifier": "string (required)",
  "email": "string (required)"
}
```

**Flow:**
1. Decode the base64 participant self-description URL
2. Find the Participant by `selfDescriptionURL`
3. Find a UserIdentifier matching:
   - `attachedParticipant`: found participant's ID
   - `identifier` or `email`: matching the provided values
4. Check if a User is linked
5. Return existence status

**Response:** `200 OK` with `{ exists: boolean, userIdentifier: object | null }`

#### 4. Inject UserIdentifier IDs into User — `POST /users/identifier`

**Auth:** `verifyUserJWT`

**Request Body:**
```json
{
  "identifiers": ["userIdentifierId1", "userIdentifierId2"]
}
```

**Flow:**
1. Get authenticated user
2. For each identifier ID:
   a. Verify the UserIdentifier exists
   b. Check it's not already in the user's `identifiers` array (use `.toString()` for comparison, fixing original ObjectId comparison bug)
   c. Add to user's `identifiers` array
3. Save user
4. Return updated user

**Response:** `200 OK`

#### 5. Update UserIdentifier — `PUT /users/:id`

**Auth:** `verifyParticipantJWT`

**Request Body:**
```json
{
  "email": "string (optional)",
  "identifier": "string (optional)",
  "url": "string (optional)"
}
```

**Flow:**
1. Find UserIdentifier by ID
2. Verify the requesting participant owns this UserIdentifier
3. Update fields
4. Return the updated document (using `{ new: true }`)

**Response:** `200 OK` with updated UserIdentifier

#### 6. Delete UserIdentifier — `DELETE /users/:id`

**Auth:** `verifyParticipantJWT`

**Flow:**
1. Find UserIdentifier by ID
2. Verify the requesting participant owns this UserIdentifier
3. Remove the UserIdentifier's ID from the linked User's `identifiers` array (fixing orphan reference bug)
4. Delete the UserIdentifier document
5. Return success

**Response:** `200 OK`

### Cross-Participant User Federation (`checkUserIdentifier`)

**File:** `src/utils/checkUserIdentifier.ts`

This is the core federation logic that links users across participants:

```
function checkUserIdentifier(userIdentifier: IUserIdentifier): Promise<void>
```

**Algorithm:**
1. Find all UserIdentifiers with the same `email` (across all participants)
2. Find all Users linked to those UserIdentifiers (via `UserIdentifier.user` or `User.identifiers`)
3. If exactly one existing User is found:
   a. Link the new UserIdentifier to that User (`userIdentifier.user = user._id`)
   b. Add the new UserIdentifier to the User's `identifiers` array (deduplicating with `.toString()`)
   c. Save both documents
4. If no existing User is found:
   - The UserIdentifier remains unlinked (will be linked when the user signs up or when another path creates the User)
5. If multiple Users are found:
   - Log a warning (indicates data inconsistency)
   - Link to the most recently created User

### `TryFindMatchingUser` Utility

**File:** `src/libs/user-reconciliation/index.ts`

Helper function used by `checkUserIdentifier` and auth middleware:

```
function TryFindMatchingUser(email: string): Promise<IUser | null>
```

Finds a User by email matching across UserIdentifiers and direct User email lookup.

### Files to Create/Modify

```
src/controllers/usersController.ts          # Add identifier management functions
src/routes/users.ts                         # Add identifier routes
src/utils/checkUserIdentifier.ts            # Federation logic
src/libs/user-reconciliation/index.ts       # TryFindMatchingUser
```

## Acceptance Criteria

1. [ ] `POST /users/register` creates a UserIdentifier linked to the authenticated participant
2. [ ] `POST /users/register` triggers cross-participant federation via `checkUserIdentifier()`
3. [ ] `POST /users/registers` creates multiple UserIdentifiers and runs federation for each
4. [ ] `POST /users/identifier/search` finds UserIdentifiers by participant URL + identifier/email
5. [ ] `POST /users/identifier` adds UserIdentifier IDs to the authenticated user's `identifiers` array
6. [ ] `PUT /users/:id` updates a UserIdentifier owned by the authenticated participant
7. [ ] `DELETE /users/:id` removes the UserIdentifier and cleans up User references
8. [ ] `checkUserIdentifier()` correctly links a new UserIdentifier to an existing User by email
9. [ ] `checkUserIdentifier()` handles the case where no matching User exists
10. [ ] ObjectId comparisons use `.toString()` or `.equals()`, not `===`
11. [ ] Integration tests cover single registration, bulk registration, federation scenarios, and CRUD

## Known Issues to Address

1. **Bulk registration skips federation** (original bug 17.3): Run `checkUserIdentifier()` for every UserIdentifier in the bulk array.
2. **ObjectId comparison with `Array.includes`** (original bug 17.4): Use `.toString()` comparison or Mongoose `.equals()` for all ObjectId comparisons.
3. **Update returns stale data** (original bug 17.5): Use `{ new: true }` option with `findByIdAndUpdate`.
4. **Delete orphans references** (original bug 17.6): When deleting a UserIdentifier, remove its ID from the linked User's `identifiers` array.
5. **`user` field on UserIdentifier unused** (original bug 17.10): Actively set the `user` field during federation in `checkUserIdentifier()`.
