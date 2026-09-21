# TICKET-015: API Docs and Infrastructure

## Metadata

| Field | Value |
|-------|-------|
| **Ticket ID** | TICKET-015 |
| **Title** | API Docs and Infrastructure |
| **Priority** | High |
| **Complexity** | M (Medium) |
| **Dependencies** | TICKET-001 |
| **Related User Stories** | US-SM-005, US-SM-006, US-SM-007, US-SM-008, US-SM-009, US-SM-011 |

## Description

Implement the API documentation (Swagger/OpenAPI), health check endpoint, Winston logging configuration, Joi request validation middleware, global error handling, and JSON-LD self-description generation. These are cross-cutting infrastructure concerns that support all other modules.

## Technical Details

### 1. Swagger/OpenAPI Documentation

**File:** `docs/swagger.json`

**Mount Point:** `GET /docs` via `swagger-ui-express`

**Implementation:**
- Create an OpenAPI 3.0 specification documenting all API endpoints
- Include request/response schemas, authentication requirements, and examples
- Serve the spec via Swagger UI Express middleware
- The spec should be the source of truth for all endpoint contracts

**Setup in `server.ts`:**
```typescript
import swaggerUi from 'swagger-ui-express';
import swaggerDocument from '../docs/swagger.json';

app.use('/docs', swaggerUi.serve, swaggerUi.setup(swaggerDocument));
```

### 2. Health Check

**Endpoint:** `GET /health`

**Auth:** None (public)

**Response:**
```json
{
  "status": "ok",
  "version": "1.2.0",
  "uptime": 12345,
  "mongodb": "connected"
}
```

**Implementation:**
- Check MongoDB connection state
- Return service version from `package.json`
- Return process uptime

### 3. Winston Logging

**File:** `src/libs/loggers/index.ts`

**Configuration:**
- **Console transport:** Always active in development, with colorized output
- **Daily rotate file transport:** Active in production
  - `datePattern: 'YYYY-MM-DD'`
  - `maxSize: '20m'` (configurable)
  - `maxFiles: '14d'` (configurable)
  - Log directory: `logs/`
- **Log levels:** error, warn, info, http, debug

**Usage pattern:**
```typescript
import logger from '../libs/loggers';

logger.info('Server started', { port: 3000 });
logger.error('Database connection failed', { error: err.message });
```

### 4. Joi Request Validation Middleware

**File:** `src/middleware/joiValidation.ts`

**Implementation:**

A reusable middleware factory that validates request bodies against Joi schemas:

```typescript
function validateBody(schema: Joi.ObjectSchema) {
  return (req: Request, res: Response, next: NextFunction) => {
    const { error, value } = schema.validate(req.body, { abortEarly: false });
    if (error) {
      return next(new BadRequestError(error.details.map(d => d.message).join(', ')));
    }
    req.body = value; // Use validated/sanitized value
    next();
  };
}
```

**Joi Schema Files:**
- `src/libs/joi/participantSchemas.ts` — Participant create/update schemas
- Additional schemas as needed per module

### 5. Global Error Handler

**File:** `src/middleware/globalErrorHandler.ts`

**Implementation:**

Handle all 5 custom error classes (fixing original bug where only 2 were handled):

```typescript
function globalErrorHandler(err: Error, req: Request, res: Response, next: NextFunction) {
  if (err instanceof BadRequestError) {
    return res.status(400).json({ error: true, message: err.message });
  }
  if (err instanceof NotFoundError) {
    return res.status(404).json({ error: true, message: err.message });
  }
  if (err instanceof ForbiddenError) {
    return res.status(403).json({ error: true, message: err.message });
  }
  if (err instanceof BadEndpointError) {
    return res.status(502).json({ error: true, message: err.message });
  }
  if (err instanceof ThirdPartyError) {
    return res.status(502).json({ error: true, message: err.message });
  }

  // Generic unhandled error
  logger.error('Unhandled error', { error: err.message, stack: err.stack });
  const message = process.env.NODE_ENV === 'production' ? 'Internal server error' : err.message;
  return res.status(500).json({ error: true, message });
}
```

**Placement:** Must be the LAST middleware registered, after all routes (including simulated/development routes).

### 6. JSON-LD Self-Descriptions

**File:** `src/libs/jsonld/index.ts`

**Functions:**
- `generateUserJsonLd(user: IUser): string` — Generate a JSON-LD self-description for a user
- `generateParticipantJsonLd(participant: IParticipant): string` — Generate a JSON-LD self-description for a participant

**Implementation:**
- Use the `API_PREFIX` environment variable for URLs (not hardcoded `/v1`)
- Include relevant fields: name, email, type, URL

### 7. Simulated Contract Service

**File:** `src/simulated/contract/index.ts`

**Purpose:** Development-only mock of the Contract Service

**Endpoints:**
- `GET /contracts/:id` — Return a simulated contract
- `GET /contracts/bilateral` — Return simulated bilateral contracts
- `GET /contracts/ecosystem` — Return simulated ecosystem contracts

**Mounting:** Only in development mode, mounted BEFORE the global error handler (fixing original bug 13.1).

### Files to Create/Modify

```
docs/
  swagger.json                          # OpenAPI 3.0 specification
src/libs/
  loggers/index.ts                      # Winston configuration
  joi/
    index.ts                            # Validation middleware factory
    participantSchemas.ts               # Participant Joi schemas
  jsonld/index.ts                       # JSON-LD generators
src/middleware/
  globalErrorHandler.ts                 # Error handler (all 5 classes)
  joiValidation.ts                      # Validation middleware
src/simulated/
  contract/index.ts                     # Mock contract service
src/routes/
  health.ts                             # Health check route
```

## Acceptance Criteria

1. [ ] `GET /docs` serves Swagger UI with the OpenAPI spec
2. [ ] `GET /health` returns service status with MongoDB connection state
3. [ ] Winston logs to console in development and to rotating files in production
4. [ ] Joi validation middleware rejects invalid payloads with descriptive errors
5. [ ] Global error handler maps all 5 custom error classes to correct HTTP statuses
6. [ ] Global error handler suppresses stack traces in production
7. [ ] JSON-LD generators use `API_PREFIX` from environment (not hardcoded)
8. [ ] Simulated contract service is mounted before the global error handler
9. [ ] Simulated contract service is only active in development mode
10. [ ] Unit tests verify error handling, validation, and health check

## Known Issues to Address

1. **Error classes not handled** (original bug 13.2): Handle all 5 classes (`ForbiddenError`, `ThirdPartyError`, `BadEndpointError` in addition to `BadRequestError` and `NotFoundError`).
2. **Simulated router escapes error handler** (original bug 13.1): Mount the simulated router before `globalErrorHandler`.
3. **JSON-LD URL hardcodes API prefix** (original bug 13.6): Read `API_PREFIX` from environment.
4. **Simulated contract filtering bug** (original bug 13.5): Fix the filter logic to check `assigner` for provider and `assignee` for consumer (not `assignee` for both).
