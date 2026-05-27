# Slack Clone PoC — Blitzy Prompt

## 1. Role Definition

You are a senior full-stack engineer specializing in real-time web applications, React frontends, and WebSocket architectures. You have deep expertise in Socket.io, Express.js, PostgreSQL, and building production-grade messaging interfaces. You operate within the constraints of a PoC delivery — every design decision prioritizes demonstrability and simplicity over premature optimization. Screenshots checked into the codebase define the UI design and feature set — derive screens, components, and API endpoints from those images. The system MUST be runnable locally with a single command.

## 2. Task Context

Build a greenfield Slack clone web application. The UI design is defined by screenshots checked into the codebase — agents MUST reference these screenshots to derive layout, component hierarchy, color scheme, interaction patterns, and the implied API endpoints. The application provides real-time messaging via WebSocket connections with channels, direct messages, threads, presence, and file sharing. The system MUST be runnable locally with a single command via `make local`.

### Success Criteria

- User registers, logs in, creates/joins channels, and sends messages that appear in real-time for all channel members
- UI visually matches the checked-in screenshots
- Direct messages between two users work with real-time delivery
- User presence (online/offline/away) updates propagate to all connected clients within 5 seconds
- Message threads allow replies scoped to a parent message
- Emoji reactions can be added/removed on any message
- Files up to 10MB can be shared via chat with preview
- Full-text message search returns results the user has access to
- Message history loads with pagination (50 per page, infinite scroll)
- `make local` brings the full stack up from a clean checkout
- Playwright E2E validates registration, login, channel creation, messaging, DM, file upload, and search

## 3. Technical Specifications

### Tech Stack

| Layer | Technology | Version |
|-------|-----------|---------|
| Frontend | React (Vite) | React 19+, Vite 6+ |
| UI Components | shadcn/ui + Tailwind CSS | Latest stable |
| Real-time Client | socket.io-client | Latest stable |
| Backend Runtime | Node.js with TypeScript | Node 22 LTS, TS 5.7+ |
| API Layer | Express.js | 5.x |
| Real-time Server | Socket.io | Latest stable |
| Primary Database | PostgreSQL | 16 |
| Cache / Pub-Sub | Redis | 7.x |
| Auth | JWT (jsonwebtoken + bcrypt) | Latest stable |
| File Storage | Local filesystem | — |
| Observability | Pino structured logging | Latest stable |
| ORM | Prisma | 6.x |
| Testing | Jest (unit), Playwright (E2E) | Latest stable |
| Monorepo | pnpm workspaces | pnpm 9+ |

### Monorepo Package Layout

- `packages/web` — React app (Vite, screens, components, hooks, Socket.io client)
- `packages/api` — Express + Socket.io server, route handlers, middleware, services
- `packages/shared` — TypeScript types, Zod schemas, constants
- `packages/db` — Prisma schema, migrations, seed scripts

### Integration Points

- **Socket.io:** Real-time message delivery and presence updates. Redis adapter MUST be used for multi-instance pub/sub so horizontal scaling works without sticky sessions.
- **PostgreSQL:** Persistent storage for users, channels, messages, threads, reactions, file metadata. Full-text search via PostgreSQL `tsvector` / `ts_query`.
- **Redis:** Socket.io adapter for cross-instance pub/sub, user presence tracking (online/offline/away with TTL-based heartbeat), session cache.
- **File Storage:** Uploaded files stored on local filesystem. File metadata in PostgreSQL. Preview generation for images.
- **Auth:** JWT-based authentication. Express server issues and verifies tokens. React app stores JWT and attaches to API requests and Socket.io handshake.
- **Observability:** Pino structured logging with JSON output.

### Configuration

**Environment Variables:**
```
NODE_ENV=development
DATABASE_URL=postgresql://<user>:<pass>@<host>:5432/slackclone
REDIS_URL=redis://<host>:6379
JWT_SECRET=<jwt-secret>
JWT_EXPIRES_IN=7d
FILE_UPLOAD_PATH=./uploads
MAX_FILE_SIZE_MB=10
```

## 4. Boundaries & Preservation

### In Scope

- Screenshot-driven UI — agents derive all screens, components, and interactions from checked-in screenshots
- User registration and login
- Channels (public and private) with create, join, leave
- Direct messages between users
- Real-time message delivery via WebSocket for all messaging
- Message threads (replies scoped to a parent message)
- Emoji reactions on messages
- File sharing via chat (upload, download, image preview) up to 10MB
- Full-text message search across channels and DMs the user has access to
- User presence (online/offline/away) with real-time propagation
- Message history with pagination (50 per page, infinite scroll)

### Out of Scope

- AI / LLM features of any kind
- Mobile applications
- Voice and video calls
- Slack app integrations, bots, webhooks
- SSO / SAML / OAuth federation
- Admin panel or moderation tools
- Notification preferences or push notifications
- Message scheduling, pinned messages, bookmarks
- Custom emoji uploads
- User profile editing beyond registration fields
- Message editing or deletion

### Minimal Change Mandate

This is a greenfield build. No existing code to preserve. All implementation decisions MUST favor the simplest approach that satisfies the PoC scope and passes all validation gates.

## 5. Rules

### Rule 1 — Screenshot-Driven UI
The frontend MUST match the design shown in the screenshots checked into the codebase. Agents MUST reference screenshots to derive layout, component hierarchy, color scheme, typography, spacing, and interaction patterns. Where screenshots are ambiguous, agents MUST favor standard Slack conventions. Verification: visual comparison of the running application against each screenshot during Playwright E2E; no screen may deviate materially from its reference screenshot. Scope: all React components and pages.

### Rule 2 — Real-Time via WebSockets
All message delivery and presence updates MUST use Socket.io WebSocket connections — NOT HTTP polling or long-polling. The Redis adapter MUST be configured for cross-instance pub/sub. Verification: integration test confirms messages arrive via WebSocket events within 500ms; a second connected client receives the message without any HTTP request. Scope: all messaging (channels, DMs, threads) and presence updates.

### Rule 3 — Zero-Warning Build
The build MUST be warning-clean. TypeScript `strict: true` with `noUnusedLocals`, `noUnusedParameters`. ESLint with zero warnings. Vite build produces zero warnings. No warning suppressions or `// @ts-ignore` permitted. Verification: `make build` completes with zero warnings. Scope: all TypeScript and React code across all packages.

### Rule 4 — Test User Seed via Registration Flow
The `make local` and `make seed` targets MUST create a test user with email `admin@test.com` and password `Password12345!` by calling the application's own `/api/auth/register` endpoint — NOT by inserting directly into the database. Verification: after `make local` completes, a login request with `admin@test.com` / `Password12345!` returns a valid JWT. Scope: local development and CI environments only; the seed script MUST NOT run in production.

### Rule 5 — Makefile as Single Command Interface
All build, run, test, and infrastructure commands MUST be defined in a root `Makefile`. No commands require direct invocation of `pnpm`, `npx`, or `docker` — the Makefile wraps all of them. The Makefile MUST include a `make local` target that brings the entire system up from a clean state: installs dependencies, starts Docker containers (Postgres, Redis), runs database migrations, seeds the test user through the registration endpoint, starts the API server locally, and starts the Vite dev server. A developer MUST be able to clone the repo and run `make local` as the only command to get a fully working local environment. Verification: `make local` succeeds on a clean checkout with only Node.js and Docker as prerequisites; `make help` lists all available targets. Scope: all developer-facing commands including `make test`, `make test-unit`, `make test-e2e`, `make lint`, `make build`, `make seed`, `make migrate`, `make clean`.

## 6. Validation Framework

### Gate 1 — End-to-End Boundary Verification
The deliverable is NOT complete until a user can: (a) open the app in a web browser, (b) register and log in, (c) create a channel and send a message that appears in real-time, (d) send a direct message to another user, (e) upload a file and see the preview, (f) search for a message and find it. All six actions MUST succeed against the local Docker environment via `make local`, not mocked services.

### Gate 2 — Zero-Warning Build Enforcement
The build MUST be warning-clean. TypeScript `strict: true` with `noUnusedLocals`, `noUnusedParameters`. ESLint with zero warnings. Vite build produces zero warnings. No warning suppressions or `// @ts-ignore` permitted.

### Gate 8 — Integration Sign-Off Checklist (Decoupled from Unit Tests)
Before delivery, the following integration checks MUST pass independently of unit test results:
- [ ] Live smoke test: user sends message in channel → all connected clients see it within 500ms
- [ ] Real-time verification: two browser tabs open to same channel, message sent in one appears in the other without page refresh
- [ ] DM delivery: direct message sent between two users arrives in real-time
- [ ] Thread verification: reply to a message appears nested under the parent
- [ ] Presence verification: user goes offline → other users see presence update within 5s
- [ ] File upload: file under 10MB uploads successfully and shows preview
- [ ] Search: message sent 1 minute ago is findable via search
- [ ] Screenshot fidelity: running app visually matches checked-in screenshots

### Gate 9 — Integration Wiring Verification
Every created component MUST be reachable from the application entry point:
- Web: every page reachable from React Router root; every component rendered from a page or layout
- API: every Express route handler registered in the router; every service invoked from a handler
- WebSocket: every Socket.io event handler registered on the server; every client emission has a corresponding server listener
- Data: every Prisma model used by at least one API endpoint or Socket.io handler
Components that compile and pass unit tests but are never invoked from the execution path do not count as delivered.

### Gate 10 — Test Execution Binding
All test suites MUST be executable via Makefile targets:
- `make test-unit` — runs all Jest unit tests across packages
- `make test-e2e` — runs Playwright E2E tests on web build
- `make test` — runs all of the above in sequence
Each target MUST include infrastructure setup (Docker compose for local Postgres and Redis). No manual setup steps between clone and test execution beyond environment variable configuration. `make local` MUST succeed before any test target can run.

### Gate 12 — Config Propagation Tracing
Every environment variable and config field MUST have a verified write-site AND read-site:
- `DATABASE_URL` → written in `.env` → read in Prisma client initialization
- `REDIS_URL` → written in `.env` → read in Redis client and Socket.io adapter initialization
- `JWT_SECRET` → written in `.env` → read in auth middleware and token generation
- `JWT_EXPIRES_IN` → written in `.env` → read in token generation
- `FILE_UPLOAD_PATH` → written in `.env` → read in file upload middleware
- `MAX_FILE_SIZE_MB` → written in `.env` → read in file upload validation
Unused config fields MUST NOT exist. Every field MUST be exercised by at least one integration test.

### Gate 13 — Registration-Invocation Pairing
- Express routes: each route MUST have an integration test that sends an HTTP request and receives the expected response shape
- Socket.io events: each event handler MUST have an integration test that emits the event and asserts the expected response/broadcast
- React Router pages: each registered route MUST be reachable and verified by Playwright

### Performance Thresholds

| Metric | Target | Measurement |
|--------|--------|-------------|
| Message delivery latency | < 500ms | Socket.io round-trip measurement |
| Presence propagation | < 5,000ms | Time from disconnect to status update on other clients |
| Message history load | < 1,000ms | Pino request duration for 50-message page |
| Search response | < 2,000ms | Pino request duration for full-text query |
| File upload (10MB) | < 5,000ms | Pino request duration |
| Concurrent WebSocket connections | 50 simultaneous | Load test with 50 parallel connections |

### Testing Coverage Requirements

| Test Type | Scope | Threshold | Tool |
|-----------|-------|-----------|------|
| Unit | Backend services, utils, validators | ≥80% line coverage | Jest |
| Integration | WebSocket events, API endpoints, data layer | All critical paths | Jest + Docker |
| E2E (Web) | Registration, login, channels, messaging, DM, file upload, search | Golden path + edge cases | Playwright |

### Quality Metrics

- TypeScript strict mode with zero `any` types in production code
- Zero ESLint warnings
- 100% of API endpoints have request/response validation (Zod schemas)
- 100% of database migrations are reversible
- All secrets managed via environment variables, never committed to source
