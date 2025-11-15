# WARP.md

This file provides guidance to WARP (warp.dev) when working with code in this repository.

## Project overview

This repository is a Node.js (ESM) Express API for an "acquisitions" service. It uses:
- Express for the HTTP server and routing
- Neon + Drizzle ORM for PostgreSQL access
- Zod for request validation
- Winston + Morgan for structured logging
- JWT + HTTP-only cookies for authentication

## Common commands

All commands assume the working directory is the repo root.

### Install dependencies

- Install Node dependencies:
  - `npm install`

### Run the API in development

- Start the dev server with automatic reload:
  - `npm run dev`
  - Entry point: `src/index.js` → `src/server.js` → `src/app.js`
  - Listens on `process.env.PORT` or `3000` by default.

### Linting and formatting

- Lint the codebase:
  - `npm run lint`
- Lint and auto-fix issues where possible:
  - `npm run lint:fix`
- Check formatting with Prettier:
  - `npm run format:check`
- Auto-format the codebase:
  - `npm run format`

### Database (Drizzle + Neon/PostgreSQL)

Drizzle is configured via `drizzle.config.js` and models under `src/models/*.js`. These commands assume `DATABASE_URL` is set in `.env`.

- Generate Drizzle SQL from the schema:
  - `npm run db:generate`
  - Outputs to the `drizzle/` directory.
- Run pending migrations:
  - `npm run db:migrate`
- Open Drizzle Studio:
  - `npm run db:studio`

### Tests

- There is currently no test runner configured (no `npm test` script, and no top-level `tests/` directory). ESLint is prepared for tests under `tests/**/*.js`; when a test framework is added, define an npm script (e.g. `"test"`) and document it here, including how to run a single test file.

## High-level architecture

### Entry points and HTTP surface

- `src/index.js`
  - Loads environment variables via `dotenv/config`.
  - Imports `./server.js` to start the HTTP server.
- `src/server.js`
  - Imports the Express app from `./app.js`.
  - Reads `PORT` from the environment (falls back to `3000`).
  - Calls `app.listen` and logs the local URL when the server starts.
- `src/app.js`
  - Creates the Express app and wires core middleware:
    - `helmet` for security headers
    - `cors` for cross-origin requests
    - `express.json` / `express.urlencoded` for body parsing
    - `cookie-parser` for cookie handling
    - `morgan` for HTTP access logs, writing into the shared `logger`
  - Defines basic endpoints:
    - `GET /` – basic hello and logger check
    - `GET /health` – returns status, timestamp, and uptime
    - `GET /api` – simple API health message
  - Mounts feature routes:
    - `/api/auth` → `src/routes/auth.routes.js`

### Module path aliases

The project uses Node ESM `imports` in `package.json` to create internal aliases:
- `#config/*` → `./src/config/*`
- `#controllers/*` → `./src/controllers/*`
- `#middleware/*` → `./src/middleware/*`
- `#models/*` → `./src/models/*`
- `#routes/*` → `./src/routes/*`
- `#services/*` → `./src/services/*`
- `#utils/*` → `./src/utils/*`
- `#validations/*` → `./src/validations/*`

When adding new modules in these areas, prefer using the aliases (e.g. `#services/foo.service.js`) rather than long relative paths, and keep new code under the existing directory structure so the aliases remain valid.

### Configuration and infrastructure

- `src/config/database.js`
  - Uses `@neondatabase/serverless` to create a Neon HTTP client using `process.env.DATABASE_URL`.
  - Wraps the client with `drizzle-orm/neon-http` and exports:
    - `db` – the primary Drizzle database instance used by services
    - `sql` – the underlying tagged template helper (exported for advanced queries if needed)
- `src/config/logger.js`
  - Creates a `winston` logger with:
    - JSON logging, timestamps, and stack traces
    - File transports:
      - `logs/error.lg` for errors
      - `logs/combined.log` for all logs
    - Console transport enabled when `NODE_ENV !== 'production'`, with colorized, human-readable output.
  - Exported as the shared `logger` used across the app (Express, services, controllers).
- `drizzle.config.js`
  - Points Drizzle at the schema files in `src/models/*.js`.
  - Outputs migrations and artifacts into the `drizzle/` directory.
  - Reads `DATABASE_URL` from the environment, and sets dialect to `postgresql`.

### Domain model and database access

- `src/models/user.model.js`
  - Defines the `users` table via `drizzle-orm/pg-core` with fields:
    - `id` (primary key)
    - `name`, `email` (unique), `password`
    - `role` (default `"user"`)
    - `created_at`, `updated_at` timestamps
  - This schema is the source of truth for Drizzle migrations and is used by services via the `users` export.

### Services and business logic

- `src/services/auth.service.js`
  - Password utilities:
    - `hashPassword(password)` – hashes with `bcrypt` (10 rounds), logs and throws on error.
    - `comparePassword(password, hashedPassword)` – compares passwords with `bcrypt.compare`.
  - User lifecycle:
    - `createUser({ name, email, password, role })`
      - Checks for existing user by email via Drizzle (`db.select().from(users).where(eq(users.email, email))`).
      - Hashes the password, inserts a new row, and returns a selected subset of fields.
      - Logs success and propagates domain errors (e.g. email already exists).
    - `authenticateUser({ email, password })`
      - Fetches a single user by email.
      - Verifies the password via `comparePassword`.
      - On success, returns a user DTO without the password and logs authentication; on failure, throws descriptive errors (`"User not found"`, `"Invalid password"`).

Services form the core domain logic layer and are used by controllers; they depend on `db` and model definitions, not on Express-specific objects.

### Validation and utilities

- `src/validations/auth.validation.js`
  - Defines Zod schemas:
    - `signupSchema` – validates name, email, password, and role (`"user" | "admin"`).
    - `signInSchema` – validates email and password.
- `src/utils/format.js`
  - `formatValidationError(errors)` – converts Zod errors into a human-readable string, primarily used by controllers to shape API error responses.
- `src/utils/cookies.js`
  - Provides a small abstraction over Express cookies with security-oriented defaults:
    - `httpOnly`, `sameSite: 'strict'`, and `secure` when `NODE_ENV === 'production'`.
  - Exposes helpers: `set`, `clear`, and `get` that operate on Express `req`/`res` objects.
- `src/utils/jwt.js`
  - Wraps `jsonwebtoken` for issuing and verifying JWTs using `JWT_SECRET` and `JWT_EXPIRES_IN` (defaults to 1 day).
  - Provides `jwttoken.sign(payload)` and `jwttoken.verify(token)` as the main interface for controllers and middleware.

### Controllers and routing

- `src/controllers/auth.controller.js`
  - `signup`
    - Validates request body with `signupSchema`.
    - Calls `createUser` service to persist a new user.
    - Issues a JWT via `jwttoken.sign` and writes it to an HTTP-only cookie via `cookies.set`.
    - Returns a sanitized user payload and logs the event.
  - `signIn`
    - Validates credentials with `signInSchema`.
    - Calls `authenticateUser` to verify the user.
    - Issues a JWT and sets the cookie as in `signup`.
  - `signOut`
    - Clears the `token` cookie via `cookies.clear` and logs the event.
- `src/routes/auth.routes.js`
  - Express router that wires HTTP endpoints to controller functions:
    - `POST /api/auth/sign-up` → `signup`
    - `POST /api/auth/sign-in` → `signIn`
    - `POST /api/auth/sign-out` → `signOut`

### Middleware

- `src/middleware/`
  - This directory exists but currently has no middleware implementations in the repo. It is the intended place to add shared Express middleware (e.g. auth guards, error handlers) that can be used across routes.

### Environment and configuration notes

- Environment variables
  - The app relies on `.env` (loaded via `dotenv/config`) for configuration such as:
    - `DATABASE_URL` (required by Drizzle/Neon)
    - `PORT` (optional override for the HTTP port)
    - `JWT_SECRET` (secret for JWT signing/verification)
    - `LOG_LEVEL` (optional log level override)
    - `NODE_ENV` (affects logging and cookie security)
- Logs and artifacts
  - Application logs are written under `logs/` as configured by `src/config/logger.js`.
  - Drizzle artifacts/migrations live in the `drizzle/` directory, driven by `drizzle.config.js` and the model definitions in `src/models/`.

When extending the system, prefer following the existing layering and alias patterns: expose database schema in `src/models`, route HTTP traffic through `src/routes` → `src/controllers` → `src/services`, and keep infrastructural concerns (logging, DB, config) isolated under `src/config` and `src/utils`. 