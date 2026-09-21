# TICKET-014: Consent Agent Integration

## Metadata

| Field | Value |
|-------|-------|
| **Ticket ID** | TICKET-014 |
| **Title** | Consent Agent Integration |
| **Priority** | Low |
| **Complexity** | M (Medium) |
| **Dependencies** | TICKET-001, TICKET-003 |
| **Related User Stories** | US-SM-001, US-SM-002, US-SM-003 |

## Description

Integrate the `contract-agent` package (from the Prometheus-X contract-consent-agent repository) which provides AI-powered consent and data exchange recommendations, user preference management, and configuration endpoints. This is an optional module that is dynamically loaded when a configuration file exists.

## Technical Details

### Package: `contract-agent`

**NPM Package:** `contract-agent` (external Prometheus-X package)

**Configuration File:** `consent-agent.config.json` (project root)

The consent agent is conditionally loaded — if the configuration file does not exist, the module is skipped and the endpoints return 404 or are not mounted.

### Endpoints

All endpoints are served under the consent agent's own router, typically mounted at the API prefix.

#### Profile Management

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| `GET` | `/profile` | `verifyParticipantJWT` | Get user profile for recommendations |
| `PUT` | `/profile` | `verifyParticipantJWT` | Update user profile |

#### Preferences

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| `GET` | `/profile/preferences` | `verifyParticipantJWT` | Get user preferences |
| `POST` | `/profile/preferences` | `verifyParticipantJWT` | Create/update preferences |
| `DELETE` | `/profile/preferences/:id` | `verifyParticipantJWT` | Delete a preference |

#### Configurations

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| `GET` | `/configurations` | `verifyParticipantJWT` | Get agent configurations |
| `PUT` | `/configurations` | `verifyParticipantJWT` | Update configurations |

#### Recommendations

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| `GET` | `/consents/recommendations` | `verifyParticipantJWT` | Get consent recommendations based on preferences |
| `GET` | `/data-exchanges/recommendations` | `verifyParticipantJWT` | Get data exchange recommendations |

#### Matching

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| `POST` | `/preferences/match` | `verifyParticipantJWT` | Match preferences against available consents |

### Dynamic Loading

```typescript
import { existsSync } from 'fs';
import { resolve } from 'path';

const CONFIG_PATH = resolve(process.cwd(), 'consent-agent.config.json');

export function mountConsentAgent(app: Express): void {
  if (!existsSync(CONFIG_PATH)) {
    console.log('Consent agent config not found, skipping...');
    return;
  }

  // Dynamic import to avoid loading the package if not configured
  const { ConsentAgent } = require('contract-agent');
  const agent = new ConsentAgent(CONFIG_PATH);
  // Mount agent routes
}
```

### Configuration File Format

**File:** `consent-agent.config.json` (see `consent-agent.config.sample.json`)

```json
{
  "source": "mongodb",
  "url": "mongodb://...",
  "dbName": "consent-manager",
  "agentConfig": {
    "recommendations": { "enabled": true },
    "matching": { "enabled": true }
  }
}
```

### Files to Create/Modify

```
src/libs/consentAgent/
  index.ts                        # Dynamic loader and route mounter
  consentAgentRoutes.ts           # Route definitions
src/routes/index.ts               # Conditionally mount consent agent routes
consent-agent.config.sample.json  # Sample configuration
```

## Acceptance Criteria

1. [ ] Consent agent routes are mounted only when `consent-agent.config.json` exists
2. [ ] When config is missing, the application starts normally without errors
3. [ ] Profile CRUD endpoints work with participant JWT authentication
4. [ ] Preferences CRUD endpoints work with participant JWT authentication
5. [ ] Recommendation endpoints return appropriate responses
6. [ ] Configuration endpoints allow agent configuration management
7. [ ] Dynamic loading does not crash the application if the `contract-agent` package is not installed
8. [ ] Integration tests verify conditional loading behavior

## Known Issues to Address

1. **External package dependency**: The `contract-agent` package is an external dependency that may change. Pin the version and document the expected API surface.
2. **Route definitions external**: The actual route handlers are defined within the `contract-agent` package, not in this codebase. Document the expected interface clearly.
3. **No graceful degradation**: If the consent agent fails during initialization, it should log a warning and continue without it, not crash the application.
