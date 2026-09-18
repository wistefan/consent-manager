# Prometheus-X Consent Manager

## Overview
A TypeScript/Express.js service (v1.2.0) for managing consent in the Prometheus-X data space ecosystem. It handles consent lifecycle management, user identity federation across participants, privacy notice generation from contracts, encrypted data exchange orchestration, and participant (provider/consumer) registration.

## Tech Stack
- Language: TypeScript (Node.js)
- Build: pnpm, tsc (TypeScript compiler)
- Framework: Express.js 4.18
- Database: MongoDB 7+ (Mongoose 7.3, requires replica set)
- Test: Mocha + Chai + Supertest + Sinon + nock (ts-mocha runner, mochawesome reporter)
- Auth: JWT (jsonwebtoken), bcryptjs, express-session + connect-mongo
- Email: Nodemailer + Mailchimp/Mandrill transactional
- Logging: Winston with daily-rotate-file
- Validation: Joi
- API Docs: Swagger UI Express (OpenAPI spec at `docs/swagger.json`)
- External package: `contract-agent` (from Prometheus-X contract-consent-agent repo)

## Project Structure
```
src/
  index.ts              # Entry point - connects to MongoDB, starts Express server
  server.ts             # Express app setup (middleware, routes, session, CORS)
  routes/
    index.ts            # Route loader - mounts all sub-routers
    consent.ts          # Consent CRUD + lifecycle endpoints (~22 routes)
    users.ts            # User signup/login/profile/identifier endpoints (~12 routes)
    participants.ts     # Participant registration/login/management (~9 routes)
    dataExchange.ts     # Data exchange trigger endpoints (push user/provider, pull stub)
    oauth.ts            # OAuth authorize/token endpoints (DEAD CODE - not mounted)
  controllers/
    consentsController.ts   # Core consent logic (~20 exported + internal functions)
    usersController.ts      # User and UserIdentifier management (~10 functions)
    participantsController.ts # Participant CRUD + DSC integration (~9 functions)
    dataExchangeController.ts # Data exchange orchestration (2 functions)
  models/
    Consent/            # Consent model - lifecycle states, events, receipt generation
    User/               # User model - auth, identifiers, bcrypt password hashing
    Participant/        # Participant model - DSC endpoints, clientID/secret
    UserIdentifier/     # UserIdentifier model - cross-participant user linking
    PrivacyNotice/      # PrivacyNotice model - generated from contracts
  middleware/           # (note: directory name without 's' in some places)
    auth.ts             # verifyParticipantJWT, verifyUserKey, verifyUserJWT
    consentKeyCheck.ts  # x-catalog-consent-key header validation
    globalErrorHandler.ts # Express error handler (400/404/500)
    joiValidation.ts    # Joi schema selection and payload validation
    oauth.ts            # validateAccessToken (unused)
    objectIdFormatCheck.ts # MongoDB ObjectId format validation
    participantsMiddleware.ts # setUserIdForParticipant (resolve UserIdentifier -> User)
    session.ts          # express-session with MongoDB store
  libs/
    emails/             # Email sending (Mandrill + Nodemailer) and HTML templates
    consent-generator/  # Kantara Consent Receipt generator (reference/utility)
    user-reconcilation/ # Cross-participant user matching by email
    third-party/        # JSON-LD self-description utilities
  utils/                # Helper functions (checkUserIdentifier, etc.)
  config/
    keys/               # RSA + AES keys for consent signing/encryption
  simulated/
    contract/           # Simulated contract service router (for development)
  tests/                # Mocha test suites
docs/
  swagger.json          # OpenAPI 3.0 spec
postman/
  pmCollection.json     # Postman API collection
templates/              # Kubernetes Helm templates (deployment, service, secret)
```

## Build & Test
```bash
pnpm install                    # Install dependencies
pnpm run build                  # Compile TypeScript to dist/
pnpm run start                  # Run compiled app
pnpm run dev                    # Run with nodemon (development)
pnpm run test                   # Run Mocha test suite
pnpm run lint                   # ESLint check
pnpm run format                 # Prettier + ESLint fix
pnpm run generatePrivateKey     # Generate RSA private key for consent signing
pnpm run generatePublicKey      # Generate RSA public key from private key
pnpm run generateAES            # Generate AES key for consent encryption
```

## Key Conventions
- Express route handlers follow `(req, res, next)` pattern with try/catch and `next(error)`
- Authentication via middleware: `verifyUserJWT`, `verifyParticipantJWT`, `verifyUserKey`, `consentKeyCheck`
- Models use Mongoose with timestamps, instance methods, and virtuals
- External service URLs are base64-encoded in URL params (provider/consumer self-description URLs)
- Consent state machine: pending -> draft -> granted -> revoked/terminated/refused (with parent/child linking for re-grant)
- Environment configuration via dotenv (.env file)
- API prefix configurable via `API_PREFIX` env var (default: `/v1`)

## Important Files
- `src/controllers/consentsController.ts` - Core consent lifecycle logic (largest file, ~1200 lines)
- `src/models/Consent/Consent.model.ts` - Consent schema with 30+ fields
- `src/middleware/auth.ts` - All authentication middleware
- `docs/swagger.json` - Complete OpenAPI specification
- `.env.sample` - All environment variables documented
- `consent-agent.config.sample.json` - Contract-agent configuration
- `src/config/keys/` - RSA and AES keys for consent encryption
