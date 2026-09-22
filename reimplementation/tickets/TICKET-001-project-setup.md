# TICKET-001: Project Setup and Scaffolding

## Metadata

| Field | Value |
|-------|-------|
| **Ticket ID** | TICKET-001 |
| **Title** | Project Setup and Scaffolding |
| **Priority** | Critical |
| **Complexity** | L (Large) |
| **Dependencies** | None |
| **Related User Stories** | US-SM-006, US-SM-008, US-SM-009 |

## Description

Set up the foundational project structure for the reimplemented Consent Manager. This includes the TypeScript/Node.js project scaffold, Express.js server configuration, MongoDB connection with replica set support, Docker development environment, and CI pipeline configuration.

This ticket establishes the baseline that all other tickets build upon.

## Technical Details

### Project Initialization

- Initialize a new Node.js project with `pnpm` as the package manager
- Configure TypeScript (`tsconfig.json`) with strict mode enabled
- Set up ESLint and Prettier with the project's coding standards
- Configure path aliases if needed

### Core Dependencies to Install

| Package | Purpose |
|---------|---------|
| `express` (^4.18) | HTTP framework |
| `mongoose` (^7.3) | MongoDB ODM |
| `dotenv` | Environment variable loading |
| `winston` | Logging |
| `winston-daily-rotate-file` | Log rotation |
| `cors` | Cross-origin resource sharing |
| `helmet` | Security headers |
| `joi` | Request validation |
| `jsonwebtoken` | JWT auth tokens |
| `bcryptjs` | Password hashing |
| `express-session` | Session management |
| `connect-mongo` | MongoDB session store |
| `nodemailer` | Email sending |
| `axios` | HTTP client for external services |
| `swagger-ui-express` | API documentation UI |
| `ejs` | Template rendering |

### Dev Dependencies

| Package | Purpose |
|---------|---------|
| `typescript` | TypeScript compiler |
| `ts-node` / `tsx` | TypeScript execution |
| `nodemon` | Development auto-reload |
| `mocha` / `ts-mocha` | Test runner |
| `chai` | Assertion library |
| `supertest` | HTTP testing |
| `sinon` | Mocking/stubbing |
| `nock` | HTTP request mocking |
| `mochawesome` | Test reporter |
| `eslint` | Linting |
| `prettier` | Code formatting |

### Directory Structure to Create

```
src/
  index.ts              # Entry point
  server.ts             # Express app setup
  config/
    database.ts         # MongoDB connection
    env.ts              # Environment variable loading with defaults
  routes/
    index.ts            # Route loader
  controllers/          # (empty, created by later tickets)
  models/               # (empty, created by later tickets)
  middleware/
    globalErrorHandler.ts  # Express error handler
  errors/
    index.ts            # Custom error classes (BadRequestError, NotFoundError, ForbiddenError, BadEndpointError, ThirdPartyError)
  libs/
    loggers/            # Winston logger configuration
  types/                # TypeScript type definitions
  utils/                # Helper utilities
docker/
  app/
    Dockerfile          # Multi-stage build
  mongodb/
    Dockerfile          # MongoDB with replica set init
docs/
  swagger.json          # OpenAPI 3.0 spec skeleton
```

### Express Server Configuration (`server.ts`)

- Mount CORS middleware with configurable origin
- Mount `express.json()` and `express.urlencoded()` body parsers
- Mount `helmet` for security headers
- Mount session middleware with MongoDB store
- Mount route loader at configurable `API_PREFIX` (default: `/v1`)
- Mount Swagger UI at `/docs`
- Mount health check at `/health`
- Mount global error handler (last middleware)

### MongoDB Connection (`config/database.ts`)

- Connect using `MONGODB_URI` environment variable
- Support replica set configuration
- Handle connection events (connected, error, disconnected)
- Graceful shutdown on process signals (SIGINT, SIGTERM)

### Environment Configuration (`config/env.ts`)

Load and validate all environment variables from `.env` file:

| Variable | Default | Required | Description |
|----------|---------|----------|-------------|
| `PORT` | `3000` | No | Server port |
| `MONGODB_URI` | — | Yes | MongoDB connection string |
| `JWT_SECRET_KEY` | — | Yes | JWT signing secret |
| `OAUTH_SECRET_KEY` | — | Yes | OAuth JWT signing secret |
| `API_PREFIX` | `/v1` | No | API route prefix |
| `X_CATALOG_CONSENT_KEY` | — | Yes | Catalog service auth key |
| `CONSENT_MANAGER_URL` | — | Yes | Public URL of this service |
| `NODE_ENV` | `development` | No | Runtime environment |
| `SESSION_SECRET` | — | Yes | Express session secret |
| `MANDRILL_API_KEY` | — | No | Mandrill email API key |
| `NODEMAILER_HOST` | — | No | SMTP host |
| `NODEMAILER_PORT` | — | No | SMTP port |
| `NODEMAILER_USER` | — | No | SMTP username |
| `NODEMAILER_PASS` | — | No | SMTP password |
| `CONTRACT_SERVICE_URL` | — | No | Contract service base URL |

### Custom Error Classes (`errors/index.ts`)

Create the following error classes, each extending a base `AppError`:

| Class | HTTP Status | Use Case |
|-------|-------------|----------|
| `BadRequestError` | 400 | Invalid request data |
| `NotFoundError` | 404 | Resource not found |
| `ForbiddenError` | 403 | Insufficient permissions |
| `BadEndpointError` | 502 | External service endpoint failure |
| `ThirdPartyError` | 502 | External service returned error |

### Global Error Handler (Stub)

Create `src/middleware/globalErrorHandler.ts` as a minimal stub that catches errors and returns a generic 500 JSON response. The full implementation (handling all 5 error classes, production stack trace suppression) is owned by TICKET-015.

### Logging Configuration (Stub)

Create `src/libs/loggers/index.ts` as a minimal wrapper around `console` (or a basic Winston instance with console transport only). The full configuration (daily-rotate-file transport, configurable max size/file count, log levels) is owned by TICKET-015.

### Swagger Spec (Skeleton)

Create `docs/swagger.json` as a minimal OpenAPI 3.0 skeleton with the `info` block and an empty `paths` object. The full endpoint documentation is owned by TICKET-015.

### Docker Configuration

- Multi-stage Dockerfile for production builds
- Docker Compose with:
  - App service (Node.js)
  - MongoDB service with replica set initialization
  - Shared network
  - Volume mounts for development

### NPM Scripts

| Script | Command | Description |
|--------|---------|-------------|
| `build` | `tsc` | Compile TypeScript |
| `start` | `node dist/index.js` | Run production build |
| `dev` | `nodemon src/index.ts` | Development with auto-reload |
| `test` | `ts-mocha ...` | Run test suite |
| `lint` | `eslint src/` | Lint check |
| `format` | `prettier --write . && eslint --fix src/` | Format code |

## Acceptance Criteria

1. [ ] Running `pnpm install` succeeds with no errors
2. [ ] Running `pnpm run build` compiles TypeScript without errors
3. [ ] Running `pnpm run dev` starts the Express server and connects to MongoDB
4. [ ] `GET /health` returns `200 OK` with service status
5. [ ] `GET /docs` serves Swagger UI with the OpenAPI spec skeleton
6. [ ] All 5 custom error classes are defined and a stub global error handler is mounted (full implementation in TICKET-015)
7. [ ] A stub Winston logger is available for import (full configuration in TICKET-015)
8. [ ] `docker-compose up` starts the app and MongoDB with replica set
9. [ ] Environment variables are loaded from `.env` with sensible defaults
10. [ ] ESLint and Prettier configurations are in place and `pnpm run lint` passes
11. [ ] A `.env.sample` file documents all environment variables

## Known Issues to Address

These bugs from the original codebase should be corrected in the reimplementation:

1. **Global error handler incomplete** (original bug 13.2): The original handler only handles `BadRequestError` (400) and `NotFoundError` (404). The reimplementation must handle all 5 custom error classes.
2. **Simulated contract router escapes error handler** (original bug 13.1): Ensure all routes — including development-only routes — are mounted before the global error handler.
3. **User JSON-LD URL hardcodes API prefix** (original bug 13.6): Use `API_PREFIX` from environment configuration throughout, never hardcode `/v1`.
4. **Key files loaded synchronously at import** (original bug 18.7): Design the key loading to be lazy or async, not synchronous at module import time.
