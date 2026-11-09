# genuivo_api

Backend API server for the Genuivo application. Built with [Hono](https://hono.dev/) on the [Bun](https://bun.sh/) runtime, with PostgreSQL via [Drizzle ORM](https://orm.drizzle.team/) and Firebase authentication.

Private package (`genuivo_api`, not published to npm). Deployed as a Docker image.

## Requirements

- [Bun](https://bun.sh/) (runtime, package manager, test runner host, and bundler — do not use npm/yarn/pnpm to install)
- PostgreSQL database
- A Firebase project with a service account (for auth)
- Access to the `@sudobility` scoped packages, which resolve from GitHub Packages (see `.npmrc`)

## Setup

```bash
bun install
cp .env.example .env   # then fill in the values
```

`.npmrc` points the `@sudobility` scope at GitHub Packages (`npm.pkg.github.com`), so installing the private `@sudobility/*` dependencies requires a GitHub token with `read:packages` configured in your environment/global npm auth.

### Environment Variables

Loaded via `src/lib/env-helper.ts`. Priority: `.env.local` (highest) > `.env` / system env > built-in default. See `.env.example` for a complete, commented template.

| Variable | Description | Default |
|----------|-------------|---------|
| `DATABASE_URL` | PostgreSQL connection string | required |
| `FIREBASE_PROJECT_ID` | Firebase project ID | required |
| `FIREBASE_CLIENT_EMAIL` | Firebase service account email | required |
| `FIREBASE_PRIVATE_KEY` | Firebase service account private key | required |
| `SHAPESHYFT_API_URL` | ShapeShyft upstream chat endpoint | required for `/chat` |
| `SHAPESHYFT_API_KEY` | ShapeShyft API key | required for `/chat` |
| `SHAPESHYFT_TIMEOUT_MS` | Upstream chat request timeout (ms) | `120000` |
| `SITEADMIN_EMAILS` | Comma-separated admin emails (may read other users' data) | optional |
| `PORT` | Server port | `8026` |
| `NODE_ENV` | `development` \| `production` \| `test` | `development` |
| `TEST_DATABASE_URL` | DB URL for `test:db` (must be localhost; guard rejects any other host) | optional |

Note: `FIREBASE_PRIVATE_KEY` usually needs its escaped `\n` sequences turned into real newlines, depending on how it is stored in your environment.

## Commands

```bash
bun run dev            # Watch mode (bun --watch src/index.ts)
bun run start          # Run the server (bun run src/index.ts)
bun run build          # Bundle to dist/ (bun build --target bun)

bun run test:unit      # Run the Vitest test suite (colocated *.test.ts)
bun run test:watch     # Vitest in watch mode
bun run typecheck      # tsc --noEmit
bun run lint           # ESLint over src/
bun run lint:fix       # ESLint with --fix
bun run format         # Prettier --write over src/**/*.ts
bun run format:check   # Prettier --check
bun run verify         # typecheck + lint + test:unit + build (run before committing)
```

Tests run on **Vitest** (`bun run test:unit`), not Bun's built-in `bun test`.

> There is also a `test:db` script, but it references `vitest.db.config.ts`, which is not present in the repo; the script currently fails. Use `test:unit` for the working suite.

## API Routes

Base URL path prefix: `/api/v1` (except the root and health endpoints).

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| GET | `/` | Public | API metadata (name, version, status) |
| GET | `/health` | Public | Health check |
| GET | `/api/v1/histories/total` | Public | Global history total |
| POST | `/api/v1/chat` | Optional | Proxy to ShapeShyft chat; `web_search` enabled only for authenticated callers |
| GET | `/api/v1/users/:userId` | Required | Get a user's profile |
| GET | `/api/v1/users/:userId/histories` | Required | List a user's histories (paginated) |
| POST | `/api/v1/users/:userId/histories` | Required | Create a history record |
| PUT | `/api/v1/users/:userId/histories/:historyId` | Required | Update a history record |
| DELETE | `/api/v1/users/:userId/histories/:historyId` | Required | Delete a history record |

`:userId` is the caller's Firebase UID, not a database-generated ID. On authenticated routes the caller must match `:userId` unless they are a site admin. History listing supports `limit` (default 50, max 200), `offset` (default 0), and `orderBy` (`asc` | `desc`, default `desc`) query params.

All responses are wrapped in `BaseResponse<T>` built with `successResponse()` / `errorResponse()` from `@sudobility/genuivo_types`.

## Architecture

- **Auth**: Firebase Admin SDK via `@sudobility/auth_service` (token verification with caching). The `Authorization: Bearer <token>` header is verified in `src/middleware/firebaseAuth.ts`. Anonymous Firebase users are blocked (403). On the first authenticated request a user row is auto-created — there is no separate registration endpoint.
- **Database**: PostgreSQL + Drizzle ORM. Schema name `genuivo`, with `users` (keyed by `firebase_uid`) and `histories` (UUID PK, FK to users, cascade delete). Tables are created via raw SQL on startup in `src/db/schema.ts` — **there are no Drizzle migration files**.
- **DB connection**: lazy, via a `Proxy` in `src/db/index.ts` — the connection is opened on the first query, so connection errors surface then, not at startup.
- **Chat**: `/api/v1/chat` proxies to the ShapeShyft upstream service; returns 503 if `SHAPESHYFT_*` env vars are unset.

### Source Layout

```
src/
├── index.ts                  # App entry: Hono app, CORS, logger, error handler, port, DB init
├── db/
│   ├── index.ts              # Lazy DB connection (Proxy) + initDatabase()
│   └── schema.ts             # Drizzle schema + raw-SQL table creation
├── middleware/
│   └── firebaseAuth.ts       # Firebase token verification middleware
├── routes/
│   ├── index.ts              # Route aggregator (public vs. auth-required)
│   ├── users.ts              # GET user profile
│   ├── histories.ts          # Histories CRUD (paginated list)
│   ├── historiesTotal.ts     # GET /histories/total (public)
│   └── chat.ts               # POST /chat (ShapeShyft proxy)
├── services/
│   └── firebase.ts           # Firebase Admin initialization + verifyIdToken
└── lib/
    ├── env-helper.ts         # getEnv / getRequiredEnv with .env.local priority
    └── serializers.ts        # Row -> API-type serializers + validators
```

Tests are colocated as `*.test.ts` next to the code they cover.

## Docker

The `Dockerfile` builds a production image on `oven/bun`, runs as a non-root user, exposes `8026`, and health-checks `/health`. It installs private packages from `registry.npmjs.org` using a build-time `NPM_TOKEN` arg (note: this differs from the local `.npmrc`, which uses GitHub Packages). CI/CD (`.github/workflows/ci-cd.yml`) runs tests on `develop` and builds/deploys the Docker image on `main` via a reusable workflow.

## Related Projects

- **genuivo_types** — Shared TypeScript request/response types and `BaseResponse` helpers consumed here
- **genuivo_client** — API client SDK that calls this server's HTTP endpoints
- **genuivo_lib** — Business-logic library that uses genuivo_client to talk to this API
- **genuivo_app** — Web frontend (defaults to this API on port 8026)
- **genuivo_app_rn** — React Native mobile app

## License

BUSL-1.1 (private, not published — see `package.json`).
