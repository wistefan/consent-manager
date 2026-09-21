# TICKET-017: PDI Integration

## Metadata

| Field | Value |
|-------|-------|
| **Ticket ID** | TICKET-017 |
| **Title** | PDI Integration |
| **Priority** | Medium |
| **Complexity** | S (Small) |
| **Dependencies** | TICKET-008 |
| **Related User Stories** | US-CM-014 |

## Description

Implement the PDI (Personal Data Intermediary) iframe redirect endpoint that allows embedding the consent management UI in participant applications. The PDI is an external frontend application that presents consent choices to end users.

## Technical Details

### Endpoint

#### PDI Iframe Redirect — `GET /consents/pdi/iframe`

**Auth:** `verifyUserKey` or `verifyParticipantJWT`

**Query Parameters:**
- `privacyNoticeId` (required) — ID of the privacy notice to present
- `userId` (required) — User identifier
- `userIdentifier` (required) — UserIdentifier ID
- `callbackUrl` (optional) — URL to redirect after consent action

**Flow:**
1. Validate required query parameters
2. Build the PDI frontend URL with query parameters:
   ```
   {PDI_FRONTEND_URL}/consent?
     consentManagerUrl={CONSENT_MANAGER_URL}&
     privacyNoticeId={privacyNoticeId}&
     userId={userId}&
     userIdentifier={userIdentifier}&
     callbackUrl={callbackUrl}
   ```
3. Redirect the user's browser (HTTP 302) to the PDI URL

**Response:** `302 Redirect` to PDI frontend

### Environment Configuration

| Variable | Default | Description |
|----------|---------|-------------|
| `PDI_FRONTEND_URL` | — | Base URL of the PDI frontend application |
| `CONSENT_MANAGER_URL` | — | Public URL of this Consent Manager instance |

### PDI Frontend Interaction

The PDI frontend is a separate application that:
1. Receives the consent context via query parameters
2. Fetches the privacy notice from the Consent Manager API
3. Presents the consent choices to the user
4. Calls back to the Consent Manager API to grant/refuse consent
5. Redirects to the `callbackUrl` when done

### EJS Template (Alternative)

If the PDI is not available, the Consent Manager has a built-in EJS template:

**File:** `src/views/consent.ejs`

This template renders a basic consent form and can be used as a fallback when no external PDI is configured.

### Files to Create/Modify

```
src/controllers/consentsController.ts  # pdiIframeRedirect function
src/routes/consent.ts                  # PDI route
src/views/consent.ejs                  # Fallback consent template
```

## Acceptance Criteria

1. [ ] `GET /consents/pdi/iframe` redirects to the PDI frontend with correct query parameters
2. [ ] Missing `PDI_FRONTEND_URL` returns an appropriate error or uses the built-in template
3. [ ] All required query parameters are validated
4. [ ] URL parameters are properly encoded
5. [ ] The EJS consent template renders correctly as a fallback
6. [ ] Integration test verifies the redirect URL construction

## Known Issues to Address

1. **No validation of callbackUrl**: The original does not validate that the `callbackUrl` is safe (could be used for open redirect attacks). Validate against a whitelist or at minimum ensure it's a valid URL.
2. **PDI URL not configurable**: Ensure the PDI frontend URL is configurable via environment variable.
