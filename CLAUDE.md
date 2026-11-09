# Genuivo API

> **Git policy — never auto-commit or auto-push.** Leave your work in the working tree.
> Run `git commit`, `git push`, `gh pr create`, or `scripts/push_all.sh` **only when the user
> explicitly asks in that turn**. Approval for an earlier change does not carry forward, and
> finishing a task is not permission to commit it.

Backend API server for the Genuivo application.

**Package**: `genuivo_api` (private, not published)

## Tech Stack

- **Language**: TypeScript
- **Runtime**: Bun
- **Package Manager**: Bun (do not use npm/yarn/pnpm for installing dependencies)
- **Framework**: Hono
- **Database**: PostgreSQL via Drizzle ORM
- **Auth**: Firebase Admin SDK (via @sudobility/auth_service)
- **Test**: Vitest

## Project Structure

```
src/
├── index.ts                        # App entry, Hono setup, port config
├── db/
│   ├── index.ts                    # Lazy DB connection via Proxy pattern
│   └── schema.ts                   # Drizzle schema + raw SQL table creation
├── middleware/
│   └── firebaseAuth.ts             # Firebase token verification middleware
├── routes/
│   ├── index.ts                    # Route aggregator
│   ├── users.ts                    # User routes
│   ├── histories.ts                # CRUD /api/v1/users/:userId/histories (paginated list)
│   ├── historiesTotal.ts           # GET /api/v1/histories/total (public)
│   └── chat.ts                     # POST /api/v1/chat (ShapeShyft proxy)
├── services/
│   └── firebase.ts                 # Firebase Admin init + verifyIdToken
└── lib/
    ├── env-helper.ts               # Environment variable helper
    └── serializers.ts              # Row -> API-type serializers + validators
```

## Commands

```bash
bun run dev            # Watch mode (bun --watch)
bun run build          # Build (bun build --target bun)
bun run start          # Run server (bun run src/index.ts)
bun run test:unit      # Run Vitest suite (colocated *.test.ts) — NOT `bun test`
bun run test:watch     # Vitest watch mode
bun run typecheck      # tsc --noEmit
bun run lint           # Run ESLint over src/
bun run format         # Prettier --write src/**/*.ts
bun run verify         # typecheck + lint + test:unit + build (use before commit)
```

Tests run on Vitest, not Bun's built-in `bun test`. The `test:db` script is broken: it points at `vitest.db.config.ts`, which does not exist in the repo.

## API Routes

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| GET | `/` | Public | API metadata |
| GET | `/health` | Public | Health check |
| GET | `/api/v1/histories/total` | Public | Get global total |
| POST | `/api/v1/chat` | Optional | ShapeShyft chat proxy; `web_search` on only if authenticated |
| GET | `/api/v1/users/:userId` | Required | Get user profile |
| GET | `/api/v1/users/:userId/histories` | Required | List user histories (paginated: `limit`≤200, `offset`, `orderBy`) |
| POST | `/api/v1/users/:userId/histories` | Required | Create history |
| PUT | `/api/v1/users/:userId/histories/:historyId` | Required | Update history |
| DELETE | `/api/v1/users/:userId/histories/:historyId` | Required | Delete history |

## Database Schema

Schema name: `genuivo`

- **users** — `firebase_uid` (PK), `email`, `display_name`, timestamps
- **histories** — `id` (UUID PK), `user_id` (FK), `datetime`, `value` (numeric), timestamps

Tables are created via raw SQL (no migration files). The DB connection uses a Proxy pattern for lazy initialization.

## Auth Flow

1. Firebase token extracted from `Authorization: Bearer <token>` header
2. Token verified via @sudobility/auth_service (cached)
3. Anonymous users are blocked (403)
4. User auto-created in DB on first authenticated request

## Environment Variables

| Variable | Description | Default |
|----------|-------------|---------|
| `DATABASE_URL` | PostgreSQL connection string | required |
| `FIREBASE_PROJECT_ID` | Firebase project ID | required |
| `FIREBASE_CLIENT_EMAIL` | Firebase service account email | required |
| `FIREBASE_PRIVATE_KEY` | Firebase service account private key | required |
| `SHAPESHYFT_API_URL` / `SHAPESHYFT_API_KEY` | Upstream chat service (required for `/chat`) | required for chat |
| `SHAPESHYFT_TIMEOUT_MS` | Chat upstream timeout (ms) | `120000` |
| `SITEADMIN_EMAILS` | Comma-separated admin emails (can read other users' data) | optional |
| `PORT` | Server port | `8026` |

Env is read via `src/lib/env-helper.ts`: `.env.local` > `.env`/system > default. See `.env.example`.

## Related Projects

- **genuivo_types** — Shared TypeScript type definitions; this project imports request/response types and helpers from it
- **genuivo_client** — API client SDK that consumes this server's HTTP endpoints
- **genuivo_lib** — Business logic library that uses genuivo_client to talk to this API
- **genuivo_app** — Web frontend that calls this API (both default to port 8026)
- **genuivo_app_rn** — React Native app that calls this API

Uses `@sudobility/auth_service` for Firebase token verification with caching.

## Coding Patterns

- Routes are defined in separate files under `src/routes/` and aggregated in `src/routes/index.ts`
- Firebase auth middleware (`src/middleware/firebaseAuth.ts`) handles token verification with caching via `@sudobility/auth_service`
- DB connection uses a Proxy pattern for lazy initialization (only connects when first queried)
- Database tables are created via raw SQL in `src/db/schema.ts` -- there are no Drizzle migration files
- All API responses must be wrapped in `BaseResponse<T>` using `successResponse()` or `errorResponse()` from `@sudobility/genuivo_types`
- Users are auto-created in the database on their first authenticated request (no separate registration endpoint)
- Route handlers follow the pattern: extract params -> validate -> query DB -> return wrapped response

## Gotchas

- Database tables are created via raw SQL on startup, NOT via Drizzle migrations -- do not create migration files
- The default port is `8026`, matching `genuivo_app`'s default `VITE_API_URL` of `localhost:8026`
- Anonymous Firebase users are blocked with 403 -- only fully authenticated users are allowed
- The `userId` in route paths (`:userId`) is the Firebase UID, not a database-generated ID
- The DB Proxy pattern means connection errors only surface on first actual query, not at startup
- `FIREBASE_PRIVATE_KEY` often needs newline escaping (`\\n` -> `\n`) depending on how it is set in the environment
- `@sudobility/*` packages resolve from GitHub Packages (`.npmrc` -> `npm.pkg.github.com`); install needs a GitHub `read:packages` token. The Dockerfile instead installs them from `registry.npmjs.org` via an `NPM_TOKEN` build arg — the two paths differ
- `test:db` is currently broken (missing `vitest.db.config.ts`); use `test:unit`
- The schema object in `src/db/schema.ts` is named `starterSchema` and its JSDoc still says "starter", but the actual Postgres schema is `genuivo` (`pgSchema("genuivo")`). The root `/` endpoint and startup log in `src/index.ts` are also hardcoded to the string "Starter API" / version "1.0.0" — legacy naming, not the real package version

## Git Workflow

- Do not use feature branches for code changes. Always stay on the current branch.
