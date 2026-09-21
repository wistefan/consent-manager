# User Stories

This document defines user stories for every feature of the Prometheus-X Consent Manager.
Each story follows the standard format: *As a [role], I want [action], so that [benefit]*.
Stories are organized by module, prioritized, and cross-referenced to the feature
documentation where each capability is described in detail.

## Roles

| Role | Description |
|------|-------------|
| **User** | An individual whose personal data is subject to consent (the PII Principal). Interacts via the PDI frontend or API with JWT/session authentication. |
| **Participant** | An organization (data provider or consumer) registered in the data space. Authenticates via clientID/clientSecret and JWT. |
| **Catalog Administrator** | An operator of the data space catalog. Authenticates via the `x-catalog-consent-key` header. |
| **System Operator** | A DevOps or platform engineer responsible for deploying, monitoring, and maintaining the Consent Manager instance. |
| **Developer** | A software engineer integrating with or extending the Consent Manager API. |

## Priority Definitions

| Priority | Meaning |
|----------|---------|
| **Must-have** | Core functionality without which the system cannot fulfill its primary purpose. Required for MVP. |
| **Should-have** | Important functionality that significantly enhances usability or security but is not a hard blocker. |
| **Could-have** | Nice-to-have features that improve developer experience, operational convenience, or future extensibility. |

---

## 1. Consent Management User Stories

*Reference: [03-consent-management.md](./03-consent-management.md)*

### US-CM-001: Generate Privacy Notice Between Parties

**Priority:** Must-have

**Story:** As a **Participant**, I want to generate a privacy notice from a contract between a data provider and a data consumer, so that users can review the data-sharing terms before granting consent.

**Acceptance Criteria:**
1. Given valid base64-encoded provider and consumer self-description URLs, when I call the privacy-notice generation endpoint, a privacy notice is created from the fetched contract.
2. If a non-archived privacy notice already exists for the same contract, the endpoint returns the existing notice without creating a duplicate.
3. The generated privacy notice includes controller details, purposes, categories of data, recipients, retention period, and PII principal rights extracted from the contract.
4. Any previously generated privacy notice for the same parties with a different contract is archived.

*Ref: Section 2 — Privacy Notice Generation*

---

### US-CM-002: Retrieve Privacy Notice with Populated Data

**Priority:** Must-have

**Story:** As a **User**, I want to retrieve a privacy notice by its ID with fully populated data (contract details, data resources, recipients, purposes), so that I can make an informed consent decision.

**Acceptance Criteria:**
1. Given a valid privacy notice ID, the endpoint returns the notice with external data populated from the contract service.
2. The response includes the contract title, data resource descriptions, purpose descriptions, and recipient information.
3. If the privacy notice does not exist, a 404 error is returned.

*Ref: Section 2 — Privacy Notice Generation*

---

### US-CM-003: Grant Consent via User Key

**Priority:** Must-have

**Story:** As a **User** (authenticated by user key header), I want to grant consent for a privacy notice, so that a data provider and consumer can exchange my data according to the agreed terms.

**Acceptance Criteria:**
1. Given a valid privacy notice ID, user email, and provider/consumer participant IDs, a new consent record is created with status `granted` and `consented: true`.
2. If the user does not have a UserIdentifier for the provider, one is created automatically.
3. If a previously granted consent exists for the same privacy notice, the old consent becomes the parent and is set to `revoked`, and the new consent inherits the parent reference.
4. An event with type `consent given` is recorded on the consent.
5. If the user's email is unverified (no matching User account), a verification email is sent and the consent is created in `pending` status instead.

*Ref: Section 3 — Consent Grant Flow (User Key)*

---

### US-CM-004: Grant Consent via User JWT

**Priority:** Must-have

**Story:** As a **User** (authenticated by JWT), I want to grant or refuse consent for a privacy notice, so that I can manage my data-sharing permissions while logged in.

**Acceptance Criteria:**
1. Given a valid privacy notice ID and an event type (`consent given` or `refused`), a consent record is created reflecting the user's decision.
2. The authenticated user's ID is used directly; no email verification is needed.
3. If the user does not have a UserIdentifier for the provider, one is created.
4. Parent/child consent inheritance applies when re-granting for the same privacy notice.

*Ref: Section 4 — Consent Grant Flow (User JWT)*

---

### US-CM-005: Verify Consent via Email

**Priority:** Must-have

**Story:** As a **User**, I want to verify my consent by clicking a link in a verification email, so that my pending consent is confirmed and becomes active.

**Acceptance Criteria:**
1. Given a valid consent ID and token in the verification URL, the consent status transitions from `pending` to `granted`.
2. If no User account exists for the email, one is created and linked to the consent.
3. If a User account exists but lacks a UserIdentifier for the relevant participant, one is created.
4. The user is redirected to the PDI frontend after successful verification.
5. Invalid or expired tokens return an appropriate error.

*Ref: Section 5 — Email Verification Flow*

---

### US-CM-006: Revoke Consent

**Priority:** Must-have

**Story:** As a **User**, I want to revoke a previously granted consent, so that data sharing under that consent ceases immediately.

**Acceptance Criteria:**
1. Given a valid consent ID, the consent status is set to `revoked` and `consented` is set to `false`.
2. A `consent revoked` event is recorded on the consent.
3. If the consent is not in `granted` status, the operation returns an appropriate error.
4. The revocation is effective immediately; subsequent data exchange attempts for this consent are rejected.

*Ref: Section 6 — Consent Revocation*

---

### US-CM-007: Refuse Consent

**Priority:** Must-have

**Story:** As a **User**, I want to refuse a consent request, so that I can explicitly decline data sharing for a specific privacy notice.

**Acceptance Criteria:**
1. Given a valid consent ID, the consent status is set to `refused`.
2. A `consent refused` event is recorded.
3. If the consent has already been granted, refused, or revoked, the operation returns an appropriate error.

*Ref: Section 7 — Consent Refusal*

---

### US-CM-008: Terminate Consent

**Priority:** Should-have

**Story:** As a **Participant**, I want to terminate an active consent, so that I can end a data-sharing arrangement from the organizational side (e.g., contract expiry).

**Acceptance Criteria:**
1. Given a valid consent ID, the consent status is set to `terminated`.
2. A `consent terminated` event is recorded.
3. Termination can only apply to a consent that is currently `granted`.

*Ref: Section 8 — Consent Termination*

---

### US-CM-009: Re-confirm Consent

**Priority:** Should-have

**Story:** As a **User**, I want to re-confirm an existing consent, so that a data exchange can be triggered again without creating a new consent.

**Acceptance Criteria:**
1. Given a valid consent ID, a `consent re-confirmed` event is appended to the consent's event history.
2. If the `triggerDataExchange` query parameter is `true`, an inline data exchange is initiated for the re-confirmed consent.
3. The consent remains in `granted` status throughout.

*Ref: Section 9 — Consent Re-confirmation*

---

### US-CM-010: Resume Pending or Draft Consent

**Priority:** Should-have

**Story:** As a **Catalog Administrator**, I want to resume a consent that is in `pending` or `draft` status, so that the consent can transition to `granted` when external verification is completed.

**Acceptance Criteria:**
1. Given a consent in `pending` or `draft` status, the consent transitions to `granted` with `consented: true`.
2. A new UserIdentifier is created from the provided data if the user does not already have one for the participant.
3. If a `dataExchangeId` is provided, the consent record is updated with that ID.
4. Consents that are already `granted`, `revoked`, or `refused` cannot be resumed.

*Ref: Section 10 — Consent Resume*

---

### US-CM-011: List My Consents

**Priority:** Must-have

**Story:** As a **User**, I want to list all consents associated with my account with pagination support, so that I can review and manage my data-sharing permissions.

**Acceptance Criteria:**
1. The endpoint returns a paginated list of consents where the user is the subject.
2. Each consent includes the privacy notice, data provider, data consumer, status, and timestamps.
3. Pagination parameters (`limit`, `skip`) control the result set.
4. Optionally, results can be returned in consent receipt format.

*Ref: Section 12 — Consent Querying*

---

### US-CM-012: Get Consent by ID

**Priority:** Must-have

**Story:** As a **User**, I want to retrieve a specific consent by its ID, so that I can view its full details including events and receipt information.

**Acceptance Criteria:**
1. Given a valid consent ID, the endpoint returns the full consent record.
2. Optionally, the response can include the consent receipt format (ISO/IEC 29184 or Kantara KI-CR-v1.1.0).
3. If the user is not the subject of the consent, access is denied.

*Ref: Section 12 — Consent Querying*

---

### US-CM-013: Query Consents as Participant

**Priority:** Must-have

**Story:** As a **Participant**, I want to query consents for a specific user within my organization, so that I can verify which data-sharing permissions are active for that user.

**Acceptance Criteria:**
1. Given a user ID, the endpoint returns all consents where the participant is either the data provider or data consumer.
2. Pagination is supported.
3. Individual consent retrieval by ID is also supported within the participant scope.
4. If the user has no consents involving the participant, an empty list is returned.

*Ref: Section 12 — Consent Querying*

---

### US-CM-014: Redirect to PDI Iframe

**Priority:** Should-have

**Story:** As a **Participant**, I want to redirect users to the PDI (Personal Data Intermediary) frontend via an iframe URL, so that users can interact with the consent UI embedded in my application.

**Acceptance Criteria:**
1. Given valid query parameters (user identifier, provider/consumer self-description URLs), the endpoint redirects to the configured `CONSENT_MANAGER_URL` with all parameters forwarded.
2. The redirect preserves the user context so the PDI can display the correct consent flow.

*Ref: Section 13 — PDI Iframe Redirect*

---

### US-CM-015: Discover Available Exchanges as User

**Priority:** Should-have

**Story:** As a **User**, I want to discover available data exchanges that I can participate in, so that I can see what data-sharing opportunities exist for my identifiers.

**Acceptance Criteria:**
1. Given my authentication, the endpoint returns a list of bilateral and ecosystem contracts from the contract service that involve my registered participants.
2. The response distinguishes between contracts where I am on the provider side and those where I am on the consumer side.
3. Each exchange entry includes the contract details and participating organizations.

*Ref: Section 14 — Available Exchanges*

---

### US-CM-016: Discover Available Exchanges as Participant

**Priority:** Should-have

**Story:** As a **Participant**, I want to discover available data exchanges for my organization, so that I can see which contracts are active for a given role (provider or consumer).

**Acceptance Criteria:**
1. Given my authentication and a role (`provider` or `consumer`), the endpoint returns contracts where my organization participates in that role.
2. The response includes both bilateral and ecosystem contracts from the contract service.

*Ref: Section 14 — Available Exchanges*

---

### US-CM-017: Attach Data Exchange Token to Consent

**Priority:** Must-have

**Story:** As a **Participant** (data provider), I want to attach a data exchange token to a consent, so that the consumer can later verify the token during the data exchange flow.

**Acceptance Criteria:**
1. Given a consent ID and a token payload, the token is stored on the consent record.
2. The encrypted consent payload is then forwarded to the consumer's `consentImport` endpoint.
3. If the consent does not exist, a 404 error is returned.

*Ref: Section 15 — Token Management*

---

### US-CM-018: Validate Data Exchange Token

**Priority:** Must-have

**Story:** As a **Participant** (data consumer), I want to validate a data exchange token against a consent, so that I can verify the legitimacy of a data exchange before receiving data.

**Acceptance Criteria:**
1. Given a consent ID and a token, the endpoint checks whether the token matches the one stored on the consent.
2. If valid, the endpoint returns the associated consent and data exchange identifier.
3. If invalid, an appropriate error is returned.

*Ref: Section 15 — Token Management*

---

### US-CM-019: Generate Consent Receipt (ISO/IEC 29184)

**Priority:** Should-have

**Story:** As a **User**, I want to receive a standardized consent receipt in ISO/IEC 29184 format, so that I have a portable and interoperable record of my consent.

**Acceptance Criteria:**
1. The receipt includes: version, jurisdiction, consent timestamp, collection method, consent ID, PII principal ID, PII controller details, policy URL, services, sensitive data flag, and consent items with purposes, data categories, and legal basis.
2. The receipt is generated from the consent's `toReceipt()` instance method.
3. The format follows the ISO/IEC 29184 standard structure.

*Ref: Section 11 — Consent Receipt Formats*

---

### US-CM-020: Encrypt Consent for Secure Transmission

**Priority:** Must-have

**Story:** As the **System**, I want to encrypt consent payloads using AES-256-CBC with RSA key wrapping before transmitting them to Data Space Connectors, so that consent data is protected in transit.

**Acceptance Criteria:**
1. The consent payload is encrypted with a random AES-256-CBC key.
2. The AES key is encrypted with the RSA public key.
3. The encrypted payload includes the IV, encrypted key, and encrypted data.
4. The recipient can decrypt using the RSA private key to recover the AES key and then decrypt the payload.

*Ref: Section 16 — Consent Encryption*

---

### US-CM-021: Inherit Parent Consent on Re-grant

**Priority:** Must-have

**Story:** As a **User**, I want my new consent to reference and replace my previous consent for the same privacy notice, so that consent history is preserved and the latest consent is authoritative.

**Acceptance Criteria:**
1. When granting a new consent for a privacy notice that has an existing granted consent, the existing consent is revoked and becomes the parent.
2. The new consent stores a reference to the parent consent ID.
3. The full consent lineage (parent chain) is queryable.

*Ref: Section 3 — Consent Grant Flow (User Key), parent/child linking*

---

### US-CM-022: Generate Privacy Notice by Contract URI

**Priority:** Must-have

**Story:** As a **Participant**, I want to generate a privacy notice directly from a contract URI, so that I can create privacy notices without requiring separate provider/consumer self-description URLs.

**Acceptance Criteria:**
1. Given a base64-encoded contract URI, the endpoint fetches the contract and generates a privacy notice.
2. The generated notice includes all relevant fields (purposes, data categories, controller details).
3. If a non-archived notice already exists for that contract, the existing notice is returned.

*Ref: Section 2 — Privacy Notice Generation*

---

### US-CM-023: Trigger Inline Data Exchange on Consent Grant

**Priority:** Must-have

**Story:** As a **User**, I want to optionally trigger a data exchange immediately when I grant consent, so that data sharing begins without a separate exchange step.

**Acceptance Criteria:**
1. When `triggerDataExchange=true` is set as a query parameter during consent grant, the data exchange flow is initiated immediately after consent creation.
2. The consent record is updated with the `dataExchangeId` from the exchange.
3. If the exchange fails, the consent is still created but the error is logged.

*Ref: Section 3 — Consent Grant Flow (User Key), Section 4 — Inline Exchange*

---

### US-CM-024: Consent State Machine Enforcement

**Priority:** Must-have

**Story:** As the **System**, I want the consent lifecycle to enforce valid state transitions, so that consents cannot move to invalid states (e.g., revoking an already refused consent).

**Acceptance Criteria:**
1. Only valid state transitions are allowed: `pending` -> `granted`; `draft` -> `granted`; `granted` -> `revoked`/`terminated`; `pending`/`draft` -> `refused`.
2. Invalid transition attempts return a clear error message indicating the current state and the attempted transition.
3. Every state transition is recorded as an event on the consent.

*Ref: Section 1 — Consent Lifecycle State Machine*

---

## 2. User Management User Stories

*Reference: [04-user-management.md](./04-user-management.md)*

### US-UM-001: User Signup

**Priority:** Must-have

**Story:** As a **User**, I want to create an account with my name, email, and password, so that I can authenticate and manage my consents.

**Acceptance Criteria:**
1. Given a valid first name, last name, email, and password, a new User account is created.
2. The password is hashed with bcrypt before storage; the plaintext password is never persisted.
3. OAuth tokens (access and refresh) are generated and returned.
4. A JSON-LD self-description is generated for the new user.
5. Duplicate email registration returns an appropriate error.

*Ref: Section 1 — User Signup*

---

### US-UM-002: User Login

**Priority:** Must-have

**Story:** As a **User**, I want to log in with my email and password, so that I can access my account and manage my consents.

**Acceptance Criteria:**
1. Given valid credentials, a JWT access token and refresh token are returned.
2. A server-side session is created and linked to the user.
3. Invalid credentials return a 401 error without revealing which field is wrong.
4. The password is validated against the bcrypt hash.

*Ref: Section 2 — User Login*

---

### US-UM-003: User Logout

**Priority:** Must-have

**Story:** As a **User**, I want to log out of my session, so that my session is invalidated and cannot be reused.

**Acceptance Criteria:**
1. The server-side session is destroyed.
2. Subsequent requests with the old session cookie are rejected.
3. The response confirms successful logout.

*Ref: Section 3 — User Logout*

---

### US-UM-004: View My Profile

**Priority:** Must-have

**Story:** As a **User**, I want to view my profile information with refreshed tokens, so that I can see my account details and get valid authentication tokens.

**Acceptance Criteria:**
1. The endpoint returns the authenticated user's profile (name, email, identifiers).
2. Fresh access and refresh tokens are generated and returned.
3. The password hash is not included in the response.

*Ref: Section 4 — User Profile (Me)*

---

### US-UM-005: Look Up User by ID

**Priority:** Should-have

**Story:** As a **Developer**, I want to look up a user by their ID, so that I can retrieve basic user information for display or integration purposes.

**Acceptance Criteria:**
1. Given a valid user ID, the endpoint returns the user's public profile.
2. No authentication is required (public endpoint).
3. Sensitive fields (password hash) are excluded.
4. If the user does not exist, a 404 error is returned.

*Ref: Section 5 — User Lookup*

---

### US-UM-006: Search User by Email

**Priority:** Should-have

**Story:** As a **Catalog Administrator**, I want to search for a user by email address, so that I can locate user accounts for administrative purposes.

**Acceptance Criteria:**
1. Given an email address, the endpoint returns the matching user if found.
2. Authentication via catalog consent key is required.
3. If no user matches the email, an appropriate response is returned.

*Ref: Section 6 — User Search by Email*

---

### US-UM-007: Register Single UserIdentifier

**Priority:** Must-have

**Story:** As a **Participant**, I want to register a user identifier that links a user to my organization, so that the Consent Manager can associate consents with the correct user within my system.

**Acceptance Criteria:**
1. Given a user email, a participant-scoped identifier, and the participant's self-description URL, a UserIdentifier record is created.
2. If a User with the matching email exists, the UserIdentifier is linked to that user via cross-participant federation.
3. The identifier is associated with the authenticated participant.
4. Duplicate identifiers for the same participant are prevented.

*Ref: Section 7 — UserIdentifier Registration (Single)*

---

### US-UM-008: Register UserIdentifiers in Bulk

**Priority:** Should-have

**Story:** As a **Participant**, I want to register multiple user identifiers at once, so that I can efficiently onboard many users without individual API calls.

**Acceptance Criteria:**
1. Given an array of user identifier objects (each with email, identifier, and self-description URL), UserIdentifier records are created for each entry.
2. Existing identifiers are skipped; only new ones are created.
3. The response reports how many identifiers were created.
4. Cross-participant federation (matching by email) is applied for each identifier.

*Ref: Section 8 — UserIdentifier Registration (Bulk)*

---

### US-UM-009: Search for UserIdentifier

**Priority:** Should-have

**Story:** As a **Catalog Administrator**, I want to check whether a user identifier exists for a given participant, self-description URL, and email, so that I can verify user registration before initiating consent flows.

**Acceptance Criteria:**
1. Given participant ID, self-description URL, email, and user identifier, the endpoint returns whether a matching UserIdentifier exists and whether it is linked to a User.
2. Authentication via catalog consent key is required.
3. The response indicates the existence status clearly.

*Ref: Section 9 — UserIdentifier Search*

---

### US-UM-010: Inject UserIdentifier IDs into User

**Priority:** Should-have

**Story:** As a **Catalog Administrator**, I want to attach UserIdentifier IDs to an existing User record, so that I can manually link user identifiers that were created before the user account.

**Acceptance Criteria:**
1. Given a user ID and an array of UserIdentifier IDs, those identifiers are added to the user's `identifiers` array.
2. Authentication via catalog consent key is required.
3. Already-linked identifiers are not duplicated.

*Ref: Section 10 — UserIdentifier Injection*

---

### US-UM-011: Update UserIdentifier

**Priority:** Should-have

**Story:** As a **Participant**, I want to update a user identifier's details, so that I can correct or change the identifier information for a user in my organization.

**Acceptance Criteria:**
1. Given a UserIdentifier ID and updated fields, the record is updated.
2. Only the participant who owns the identifier can update it.
3. The response returns the updated UserIdentifier.

*Ref: Section 11 — UserIdentifier Update*

---

### US-UM-012: Delete UserIdentifier

**Priority:** Should-have

**Story:** As a **Participant**, I want to delete a user identifier, so that I can remove a user's association with my organization when it is no longer needed.

**Acceptance Criteria:**
1. Given a UserIdentifier ID, the record is deleted.
2. Only the participant who owns the identifier can delete it.
3. References to the deleted identifier in the User's `identifiers` array are also cleaned up.

*Ref: Section 12 — UserIdentifier Delete*

---

### US-UM-013: Cross-Participant User Federation

**Priority:** Must-have

**Story:** As the **System**, I want to automatically link user identifiers across participants when users share the same email address, so that a user's consents and identifiers are unified regardless of which participant registered them.

**Acceptance Criteria:**
1. When a UserIdentifier is registered, the system searches for existing User accounts with the same email.
2. If a match is found, the new UserIdentifier is linked to the existing User.
3. If no User exists but another UserIdentifier with the same email exists (linked to a User), the new identifier is linked to that same User.
4. The federation algorithm (checkUserIdentifier) runs automatically during registration.

*Ref: Section 13 — Cross-Participant User Federation*

---

## 3. Participant Management User Stories

*Reference: [05-participant-management.md](./05-participant-management.md)*

### US-PM-001: Register Participant

**Priority:** Must-have

**Story:** As a **Participant**, I want to register my organization with the Consent Manager, so that I can participate in data-sharing agreements and manage consents for my users.

**Acceptance Criteria:**
1. Given valid registration data (client ID, client secret, legal name, self-description URL, email, endpoints), a Participant record is created.
2. The client ID is unique; duplicate registration returns an error.
3. Input is validated against the Joi schema (legal name, self-description URL, and client ID are required).
4. If DSC (Data Space Connector) configuration is provided, the system fetches the participant's self-description, logs in to the DSC, and pushes the consent manager's public key and URI.

*Ref: Section 1 — Participant Registration*

---

### US-PM-002: Participant Login

**Priority:** Must-have

**Story:** As a **Participant**, I want to log in with my client ID and client secret, so that I can obtain a JWT for authenticating subsequent API requests.

**Acceptance Criteria:**
1. Given valid client ID and client secret, a JWT is returned with a 1-hour expiry.
2. Invalid credentials return a 401 error.
3. The JWT payload includes the participant's ID and client ID.

*Ref: Section 2 — Participant Login*

---

### US-PM-003: List All Participants

**Priority:** Should-have

**Story:** As a **Developer**, I want to retrieve a paginated list of all registered participants, so that I can discover available organizations in the data space.

**Acceptance Criteria:**
1. The endpoint returns a paginated list of participants.
2. Sensitive fields (client ID, client secret) are excluded from the response.
3. Pagination parameters (`limit`, `skip`) are supported.

*Ref: Section 3 — Participant Retrieval*

---

### US-PM-004: Get Participant by ID

**Priority:** Should-have

**Story:** As a **Developer**, I want to retrieve a specific participant by ID or client ID, so that I can look up an organization's details.

**Acceptance Criteria:**
1. Given a participant ID or client ID, the endpoint returns the participant's public information.
2. Sensitive fields are excluded.
3. If the participant does not exist, a 404 error is returned.

*Ref: Section 3 — Participant Retrieval*

---

### US-PM-005: View My Participant Profile

**Priority:** Must-have

**Story:** As a **Participant**, I want to view my own organization's full profile, so that I can verify my registration details and endpoint configuration.

**Acceptance Criteria:**
1. The endpoint returns the authenticated participant's full profile including endpoints.
2. Authentication via participant JWT is required.

*Ref: Section 3 — Participant Retrieval*

---

### US-PM-006: Update Participant Endpoints

**Priority:** Must-have

**Story:** As a **Participant**, I want to update my organization's endpoint configuration (data export, data import, consent export, consent import), so that the Consent Manager can communicate with my Data Space Connector.

**Acceptance Criteria:**
1. Given updated endpoint URLs and/or dataspace endpoint, the participant record is updated.
2. The update is validated against the Joi update schema.
3. Only the participant identified by the client ID in the URL can perform the update.

*Ref: Section 4 — Participant Update*

---

### US-PM-007: Delete My Participant

**Priority:** Should-have

**Story:** As a **Participant**, I want to delete my organization's registration, so that I can deregister from the Consent Manager.

**Acceptance Criteria:**
1. The authenticated participant's record is deleted.
2. Authentication via participant JWT is required.
3. Associated consents and user identifiers should be handled appropriately (note: cascade deletion is a known gap).

*Ref: Section 5 — Participant Deletion*

---

### US-PM-008: Retrieve Consent Signature Public Key

**Priority:** Must-have

**Story:** As a **Participant**, I want to retrieve the Consent Manager's public key for consent signature verification, so that I can verify the authenticity of encrypted consent payloads.

**Acceptance Criteria:**
1. The endpoint returns the RSA public key used for consent encryption.
2. The key is in PEM format.
3. No authentication is required (the key is public).

*Ref: Section 6 — Public Key Management*

---

### US-PM-009: Configure DSC Integration on Registration

**Priority:** Should-have

**Story:** As a **Participant**, I want the Consent Manager to automatically configure my Data Space Connector during registration, so that my DSC is ready to participate in consent-mediated data exchanges without manual setup.

**Acceptance Criteria:**
1. During registration, if DSC endpoints are provided, the system fetches the participant's self-description from their self-description URL.
2. The system logs in to the participant's DSC using the provided credentials.
3. The system pushes the consent manager's public key and consent URI to the DSC.
4. If any step of the DSC configuration fails, the participant is still registered but a warning is returned.

*Ref: Section 7 — DSC Integration*

---

## 4. Data Exchange User Stories

*Reference: [06-data-exchange.md](./06-data-exchange.md)*

### US-DE-001: Trigger Data Exchange as User

**Priority:** Must-have

**Story:** As a **User**, I want to trigger a data exchange for one of my granted consents, so that data is actually transferred between the provider and consumer according to my consent.

**Acceptance Criteria:**
1. Given a granted consent ID, the exchange flow is initiated: encrypt consent, send to provider's `consentExport` endpoint, receive `dataExchangeId`.
2. The consent record is updated with the `dataExchangeId`.
3. Authentication via user JWT is required.
4. If the consent is not in `granted` status, the exchange is rejected.

*Ref: Section 2 — Standalone Exchange: User-Triggered; Section 4 — Inline Exchange*

---

### US-DE-002: Trigger Data Exchange as Provider

**Priority:** Must-have

**Story:** As a **Participant** (data provider), I want to trigger a data exchange for a user who has granted consent, so that I can initiate data delivery to the consumer on behalf of the user.

**Acceptance Criteria:**
1. Given a consent ID or user identifier, the exchange flow is initiated from the provider side.
2. The user's consent is verified as `granted` before proceeding.
3. Authentication via participant JWT is required.
4. The encrypted consent payload is sent to the provider's `consentExport` endpoint.

*Ref: Section 3 — Standalone Exchange: Provider-Triggered*

---

### US-DE-003: Inline Data Exchange on Consent Grant

**Priority:** Must-have

**Story:** As the **System**, I want to automatically trigger a data exchange when a user grants consent with `triggerDataExchange=true`, so that the data flow begins immediately without a separate trigger step.

**Acceptance Criteria:**
1. When consent is granted with the `triggerDataExchange` flag, the full exchange protocol runs inline.
2. The consent record is updated with the resulting `dataExchangeId`.
3. If the exchange fails, the consent is still created but the failure is logged.
4. The inline exchange uses the same encryption and protocol as the standalone exchange.

*Ref: Section 4 — Inline Exchange (Consent Grant)*

---

### US-DE-004: Handle Token Callback from Provider

**Priority:** Must-have

**Story:** As the **System**, I want to receive a data exchange token from the data provider and forward the encrypted consent to the consumer, so that the three-party exchange protocol is completed.

**Acceptance Criteria:**
1. The provider POSTs a token to the consent's token endpoint after receiving the encrypted consent.
2. The Consent Manager stores the token on the consent record.
3. The encrypted consent payload is forwarded to the consumer's `consentImport` endpoint.
4. The consumer can then verify the token to complete the exchange.

*Ref: Section 6 — Token Callback Flow*

---

### US-DE-005: Verify Data Exchange Token

**Priority:** Must-have

**Story:** As a **Participant** (data consumer), I want to verify a data exchange token against a consent, so that I can confirm the exchange is legitimate before accepting the data.

**Acceptance Criteria:**
1. Given a consent ID and a token, the endpoint verifies the token matches the stored value.
2. If valid, the response includes the consent details and data exchange ID.
3. If invalid, a clear error is returned indicating the token does not match.

*Ref: Section 7 — Token Verification*

---

### US-DE-006: End-to-End Encrypted Data Exchange

**Priority:** Must-have

**Story:** As the **System**, I want the entire data exchange protocol to use AES-256-CBC encryption with RSA key wrapping, so that consent data is protected during all inter-service communication.

**Acceptance Criteria:**
1. A random AES-256-CBC key and IV are generated per exchange.
2. The consent payload is encrypted with the AES key.
3. The AES key is encrypted with the RSA public key.
4. The encrypted bundle (IV + encrypted key + encrypted data) is sent to the provider's DSC.
5. The provider can decrypt using the RSA private key shared during registration.

*Ref: Section 5 — Encryption Protocol; Section 9 — Complete Data Exchange Protocol*

---

## 5. Supporting Module User Stories

*Reference: [07-supporting-modules.md](./07-supporting-modules.md)*

### US-SM-001: Manage Consent Agent User Preferences

**Priority:** Should-have

**Story:** As a **User**, I want to manage my consent preferences through the Consent Agent, so that the system can recommend relevant data exchanges and pre-fill consent decisions based on my preferences.

**Acceptance Criteria:**
1. CRUD operations for user consent preferences are available through the Consent Agent endpoints.
2. Preferences are stored per user and can influence consent recommendations.
3. The Consent Agent integration uses the `contract-agent` package.

*Ref: Section 2 — Consent Agent Integration*

---

### US-SM-002: Get Consent Recommendations

**Priority:** Could-have

**Story:** As a **User**, I want to receive consent recommendations based on my preferences and available contracts, so that I can discover relevant data-sharing opportunities.

**Acceptance Criteria:**
1. The recommendation endpoint returns suggested consents based on the user's profile and preferences.
2. Recommendations are generated by the `contract-agent` package.
3. Each recommendation includes the contract, participating organizations, and data types.

*Ref: Section 2 — Consent Agent Integration*

---

### US-SM-003: Get Data Exchange Recommendations

**Priority:** Could-have

**Story:** As a **User**, I want to receive data exchange recommendations, so that I can see which of my granted consents could benefit from a data exchange.

**Acceptance Criteria:**
1. The recommendation endpoint returns suggested data exchanges for the user's granted consents.
2. Recommendations are generated by the `contract-agent` package.

*Ref: Section 2 — Consent Agent Integration*

---

### US-SM-004: Receive Consent Verification Email

**Priority:** Must-have

**Story:** As a **User**, I want to receive an email with a verification link when my consent requires email verification, so that I can confirm my identity and activate the consent.

**Acceptance Criteria:**
1. When a consent is created with an unverified email, a verification email is sent.
2. The email uses the correct HTML template (Mandrill or Nodemailer fallback).
3. The email contains a unique verification link with consent ID and token.
4. The email is sent via Mandrill transactional API; if Mandrill is not configured, the Nodemailer SMTP fallback is used.

*Ref: Section 3 — Email Service*

---

### US-SM-005: Access API Documentation

**Priority:** Should-have

**Story:** As a **Developer**, I want to access interactive API documentation, so that I can explore and test the Consent Manager's endpoints.

**Acceptance Criteria:**
1. Swagger UI is available at the `/docs` endpoint.
2. The documentation reflects the current OpenAPI specification.
3. All endpoints, request/response schemas, and authentication requirements are documented.

*Ref: Section 10 — API Documentation (Swagger)*

---

### US-SM-006: Check Service Health

**Priority:** Must-have

**Story:** As a **System Operator**, I want to check the health of the Consent Manager service, so that I can monitor availability and detect outages.

**Acceptance Criteria:**
1. The `GET /health` endpoint returns a 200 status when the service is running.
2. The response indicates the service is healthy.
3. The endpoint requires no authentication.

*Ref: Section 11 — Health Check*

---

### US-SM-007: Validate Request Payloads

**Priority:** Must-have

**Story:** As the **System**, I want to validate incoming request payloads against Joi schemas, so that malformed or invalid data is rejected before reaching the business logic.

**Acceptance Criteria:**
1. Participant creation and update requests are validated against Joi schemas.
2. Invalid payloads return a 400 error with a descriptive validation message.
3. The validation middleware runs before the controller handler.

*Ref: Section 6 — Request Validation (Joi)*

---

### US-SM-008: Handle Errors Consistently

**Priority:** Must-have

**Story:** As a **Developer**, I want the API to return consistent, structured error responses, so that I can programmatically handle errors in my integration.

**Acceptance Criteria:**
1. All errors return a JSON response with a status code and error message.
2. `BadRequestError` returns 400; `NotFoundError` returns 404; `ForbiddenError` returns 403; unhandled errors return 500.
3. The global error handler catches all unhandled exceptions.
4. MongoDB ObjectId format validation returns a clear 400 error for malformed IDs.

*Ref: Section 7 — Error Handling*

---

### US-SM-009: Configure Logging

**Priority:** Should-have

**Story:** As a **System Operator**, I want the service to produce structured, rotated log files, so that I can troubleshoot issues and maintain audit trails without running out of disk space.

**Acceptance Criteria:**
1. Logs are written to daily-rotating files with configurable maximum size and retention.
2. Five log levels are supported (error, warn, info, http, debug) with separate transports.
3. HTTP request logging via Morgan is integrated into the Winston pipeline.
4. Console output is enabled in non-production environments.

*Ref: Section 5 — Logging*

---

### US-SM-010: OAuth Authorization (Future)

**Priority:** Could-have

**Story:** As a **Developer**, I want to use OAuth 2.0 authorization flows to obtain access tokens, so that third-party applications can access the Consent Manager API with scoped permissions.

**Acceptance Criteria:**
1. An authorization endpoint accepts OAuth authorize requests with supported scopes (userRead, userWrite, organizationRead, organizationWrite).
2. A token endpoint issues access tokens after successful authorization.
3. The `validateAccessToken` middleware verifies tokens on protected endpoints.
4. *Note: This is currently dead code in the original system and needs proper implementation.*

*Ref: Section 1 — OAuth Module (Dead Code)*

---

### US-SM-011: Generate JSON-LD Self-Descriptions

**Priority:** Should-have

**Story:** As the **System**, I want to generate JSON-LD self-descriptions for users (and optionally participants), so that entities in the data space have machine-readable, standards-compliant identity representations.

**Acceptance Criteria:**
1. A schema.org Person JSON-LD self-description is generated for each user on signup.
2. The self-description includes the user's name and a unique identifier.
3. The self-description is stored on the User record and can be retrieved via the user profile endpoint.

*Ref: Section 8 — JSON-LD Self-Descriptions*

---

### US-SM-012: Manage Server Sessions

**Priority:** Must-have

**Story:** As the **System**, I want to maintain server-side sessions backed by MongoDB, so that user authentication state persists across requests and survives server restarts.

**Acceptance Criteria:**
1. Sessions are stored in MongoDB via `connect-mongo`.
2. Session cookies have a 24-hour TTL.
3. In production, cookies are configured with `secure: true` and `sameSite: 'none'`.
4. Sessions are created on login and destroyed on logout.

*Ref: Section 4 — Session Management*

---

---

## Summary

| Module | Story ID Range | Count | Must-have | Should-have | Could-have |
|--------|---------------|-------|-----------|-------------|------------|
| Consent Management | US-CM-001 to US-CM-024 | 24 | 16 | 7 | 1 |
| User Management | US-UM-001 to US-UM-013 | 13 | 6 | 7 | 0 |
| Participant Management | US-PM-001 to US-PM-009 | 9 | 5 | 4 | 0 |
| Data Exchange | US-DE-001 to US-DE-006 | 6 | 6 | 0 | 0 |
| Supporting Modules | US-SM-001 to US-SM-012 | 12 | 6 | 4 | 2 |
| **Total** | | **64** | **39** | **22** | **3** |

All 64 user stories cover the complete feature set documented in sections 03 through 07.
Each story includes a unique ID, role, action, benefit, testable acceptance criteria, priority,
and a cross-reference to the relevant feature documentation section.
