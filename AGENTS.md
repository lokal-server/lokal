# Lokal development guide

Lokal is a privacy-focused, self-hosted file-sharing application. It exposes an
existing host directory through a Next.js application while storing accounts,
shares, tags, settings, app tokens, and other metadata in PostgreSQL. Changes
must preserve that boundary: mounted files are the user's data, not disposable
application fixtures.

## Stack and architecture

- Next.js 16 App Router, React 19, and TypeScript provide the web application.
- Tailwind CSS 4 and Radix-based components provide styling and UI primitives.
- Prisma 7 with PostgreSQL stores application metadata. Files themselves stay
  on the filesystem rooted at `DATA_DIR`.
- Cookie-backed JWT sessions authenticate the main UI. The Lokal Platform API
  uses hashed, user-bound bearer tokens with explicit scopes.
- Docker Compose is the supported development and production environment. The
  production image is based on Node 20 Alpine.

Keep server-only filesystem, database, credential, and authorization work in
server actions, route handlers, or `lib/` server modules. Do not move privileged
logic into client components.

## Repository layout

- `app/` contains App Router pages, layouts, server actions' consumers, route
  handlers, the public share flow, contract-serving routes, and global styles.
  Platform routes live under `app/api/platform/`; API discovery is served from
  `app/.well-known/lokal/`.
- `components/blocks/` contains product-level UI; `components/ui/` contains
  reusable primitives. Extend existing patterns before adding new primitives.
- `lib/` contains authentication and sessions, Prisma access, filesystem and
  path helpers, platform-token handling, and shared utilities.
- `prisma/schema.prisma` defines the data model and `prisma/migrations/` contains
  committed database migrations.
- `contracts/` is the source of truth for the external Platform API OpenAPI
  document and app-manifest JSON schema.
- `scripts/` contains development and production Dockerfiles, the container
  entrypoint, image publishing, and contract drift validation.
- `docs/` contains project documentation and checked-in documentation assets.
- `proxy.ts` enforces the main session boundary for page routes. API handlers
  must perform their own authentication and authorization.

## Local development

Docker and Docker Compose are the supported prerequisites. The development
compose stack mounts the repository at `/lokal`, mounts `./test_data` at
`/data`, starts PostgreSQL and pgAdmin, and expects the external Docker network
`local-proxy` to exist. Start it with:

```sh
npm run docker:dev
```

The equivalent direct command is:

```sh
docker compose -f docker-compose.dev.yml up --build -d
```

For host-based development, use Node 20 and provide a reachable PostgreSQL
database through `DATABASE_URL` before running:

```sh
npm ci
npm run dev
```

`npm ci` runs the `postinstall` Prisma client generation. The app defaults
`DATA_DIR` to `/data`; set it to a dedicated local fixture directory when the
host process cannot or should not access `/data`. Do not point development at
real user data.

Useful environment settings are:

- `DATABASE_URL`: PostgreSQL connection string.
- `DATA_DIR`: root of the mounted files exposed by Lokal.
- `LOKAL_PUBLIC_URL`: public origin used for generated share and invite links.
- `RUN_MIGRATIONS=true`: makes the container entrypoint run
  `prisma migrate deploy` before starting the application.

## Validation

Run checks appropriate to the changed area before handoff:

```sh
npm run lint
npm run contracts:check
npm run build
```

`npm run build` already runs the contract check and Prisma client generation
before `next build`; the individual commands are useful for faster targeted
feedback. There is currently no automated test script, so do not introduce a
new test framework for a routine change. Perform focused runtime verification
when changing file operations, authentication, authorization, sharing, or API
behavior.

## Code and data rules

- Keep changes focused and follow the surrounding TypeScript, React, and route
  handler conventions. Avoid unrelated formatting or refactoring.
- Preserve the separation between filesystem content and PostgreSQL metadata.
  A schema migration must not copy mounted user files into the database or
  treat the data mount as generated application state.
- Resolve file operations beneath both `DATA_DIR` and the authenticated user's
  `rootDir`. Normalize and validate all client-provided paths and filenames;
  never permit traversal or access to another user's root.
- Keep metadata queries owner-scoped. Shares are the explicit exception and
  must continue to enforce their recipient, expiry, access, and read-limit
  rules.
- Treat Lokal Platform data as both app-scoped and user-scoped. Bearer-token
  validation must continue to check the app slug, enabled/revoked/expired
  state, required scope, `appId`, and `userId` before data access.
- Enforce roles and ownership on the server for every sensitive action and API
  route. UI visibility is not an authorization control. Avoid returning
  password hashes, raw app-token hashes, session material, or other credentials.
- Keep public endpoints deliberately narrow. Changes to `proxy.ts` exclusions,
  public shares, login/setup flows, CORS, or contract-serving routes require a
  security review of the resulting unauthenticated surface.

## Secrets and local state

- Never commit credentials, tokens, private keys, production connection
  strings, or populated environment files. Local overrides belong in ignored
  `.env*.local` files or the developer's environment.
- Compose credentials are development examples only. Do not reuse them for a
  deployed instance or add real values to compose files.
- Keep local user fixtures under the ignored `test_data/` or another directory
  outside the source tree. Never add mounted data, database volumes, or user
  uploads to Git.
- Do not log passwords, session cookies, bearer tokens, share secrets, or file
  contents. Preserve hashing and constant-time token comparison when changing
  authentication code.

## API contracts and generated artifacts

- `contracts/openapi.json` and `contracts/lokal-manifest.schema.json` are
  committed source artifacts consumed by external SDKs. Update them in the same
  change as the corresponding API or manifest behavior.
- `npm run contracts:check` compares exported HTTP methods under
  `app/api/platform/**/route.ts` and `app/.well-known/lokal/route.ts` with the
  OpenAPI document. It does not validate request or response schemas, so review
  those manually.
- For a breaking Platform API change, update `info.version` in
  `contracts/openapi.json`. Preserve discovery and contract URLs because SDKs
  consume them.
- Commit intentional Prisma schema changes and their migrations. Do not commit
  generated Prisma client files, `.next/`, `out/`, `build/`, `coverage/`,
  `next-env.d.ts`, `*.tsbuildinfo`, dependency directories, or local data.

## Deployment constraints

Production runs the published image through `docker-compose.yml`, with a host
directory mounted at `/data` and a persistent PostgreSQL volume. The entrypoint
optionally applies committed migrations before starting Next.js. Preserve this
prebuilt-image workflow so a NAS does not need a repository checkout or build
toolchain.

Image publishing uses Docker Buildx for both `linux/amd64` and `linux/arm64`.
Changes to dependencies, Prisma binary targets, Dockerfiles, the entrypoint,
compose configuration, or `scripts/push_production.sh` must remain compatible
with both architectures unless a requirement explicitly changes support. Do
not push images, deploy, or run production migrations as part of routine
development verification.
