# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Read AGENTS.md first

`AGENTS.md` is the canonical, detailed contributor guide for this repo (coding style, i18n, date/time rules, caching rules, Prisma performance rules, testing philosophy, Storybook, commit format). Treat it as authoritative for conventions. This file covers the build/test workflow and the big-picture architecture that AGENTS.md does not spell out.

Note: `AGENTS.md` ends with a Next.js docs index block stating the local `./.next-docs` is the source of truth for Next.js behavior — search those docs before relying on memory, since the app tracks a recent Next.js major.

## Commands

This is a **pnpm + Turborepo monorepo** (`pnpm@11.7.0`, Node per `engines`: 20.19+/22.13+/24). Run everything through `pnpm` from the repo root; Turbo fans tasks out across workspaces.

```bash
pnpm dev:setup          # generate .env from .env.example (fills ENCRYPTION_KEY, NEXTAUTH_SECRET, CRON_SECRET, CUBEJS_API_SECRET)
pnpm db:up              # start Postgres/Redis/etc. via docker-compose.dev.yml (db:down to stop)
pnpm go                 # db:up + run all dev servers (turbo `go`)
pnpm dev                # run all dev servers in parallel (no db:up)

pnpm build              # build all packages + apps
pnpm lint               # eslint --fix across workspace + api v3 spec lint
pnpm typecheck          # turbo typecheck (web runs next typegen first)
pnpm test               # all Vitest suites (no cache)
pnpm test:e2e           # Playwright suite (apps/web/playwright)
```

Database (the `@formbricks/database` package owns Prisma):

```bash
pnpm db:migrate:dev     # apply migrations to dev DB
pnpm fb-migrate-dev     # create a new migration + regenerate client (after schema edits)
pnpm db:seed            # seed (db:seed:clear to reset)
```

### Running a single test

Vitest runs per-package and loads env from the **root `.env`** via `dotenv`. Target the web app directly:

```bash
pnpm --filter=@formbricks/web test path/to/file.test.ts      # one file
pnpm --filter=@formbricks/web test -t "name fragment"        # by test name
```

`pnpm test` at the root has no effect without a working `.env`/DB because suites import modules that validate env. Use `pnpm dev:setup` + `pnpm db:up` first.

## Architecture

### Monorepo layout

- **`apps/web`** — the Next.js product (App Router). Everything user-facing lives here.
- **`apps/storybook`** — UI component review.
- **`packages/*`** — shared libraries consumed by `apps/web`, each its own workspace:
  - `database` — Prisma schema, migrations, and the **generated client** (`packages/database/generated`); build must run before the web app (`pnpm build --filter=@formbricks/database`).
  - `surveys` / `survey-ui` — the embeddable survey runtime (see "Survey bundle" below).
  - `cache` — Redis-backed caching (Redis is mandatory).
  - `storage` — S3-compatible file storage abstraction.
  - `types` — shared Zod schemas / TS types; prefer these over local types.
  - `ai`, `email`, `jobs`, `logger`, `i18n-utils`, `js-core`, plus `config-*` (eslint/prettier/tsconfig presets) and `vite-plugins`.

### apps/web internals

- **`app/`** — App Router entry. Route groups: `(app)` (authenticated product), `(auth)` (login/signup), `(redirects)`; plus public survey/link routes (`s/`, `c/`, `p/`), `api/`, and a trivial `health/` endpoint.
- **`modules/`** — feature domains (the bulk of the logic): `auth`, `survey`, `analysis`, `organization`, `billing`, `integrations`, `settings`, `response-pipeline`, `hub`, `storage`, `api`, `ui`, and `ee/` (Enterprise-only features). New feature work generally lives in a module, not directly in `app/`.
- **`lib/`** — cross-cutting services and helpers (e.g. `lib/env.ts`, `lib/crypto.ts`, `lib/constants.ts`).
- **Server actions** wrap service calls and consistently return `{ data }` or `{ error }`. The `cache`/`storage` packages follow the same explicit `Result<T, E>` pattern rather than throwing.

### Cross-cutting invariants

- **Env validation at build & boot.** `apps/web/lib/env.ts` (`@t3-oss/env-nextjs` + Zod) runs when `next.config.mjs` loads — during `next build` **and** `next start`. Missing/invalid required vars (`DATABASE_URL`, `ENCRYPTION_KEY`, `REDIS_URL`, `CUBEJS_API_URL/SECRET`, `HUB_API_URL/KEY`) abort the build/boot. `ENCRYPTION_KEY` must be exactly 32 chars or a 64-char hex string (`lib/crypto.ts`), else AES throws `Invalid key length`.
- **Redis is required**, not optional — `@formbricks/cache` is a `globalThis` singleton built on `REDIS_URL` and powers caching, rate limiting, and audit logging.
- **Caching:** use `cache.withCache()` / explicit Redis and always build keys with the `createCacheKey.*` utilities. **Do not** use Next.js `unstable_cache()`.
- **Multi-tenancy:** all data access must be scoped by Organization or Environment, and respect soft-delete fields (`deletedAt` / `isActive`). See the Prisma performance rules in `AGENTS.md`.
- **Testing split:** unit tests (Vitest) cover pure `.ts` logic and are colocated as `*.test.ts`; **do not** write unit tests for `.tsx` components — React UI is covered by Playwright E2E instead.

### Survey bundle (common gotcha)

`@formbricks/surveys` is **pre-compiled** (Vite → UMD/ESM) and the output is copied to `apps/web/public/js/`; the app imports from `dist/`, not source. After editing `packages/surveys` (or its deps like `survey-ui`, `types`), you **must rebuild** — and Turbo caches aggressively, so force it:

```bash
rm -rf packages/surveys/dist apps/web/public/js/surveys.* node_modules/.cache/turbo
pnpm build --filter=@formbricks/surveys... --force
```

Then hard-refresh the browser (the UMD bundle is browser-cached) and restart `pnpm dev` if changes still don't appear.

### Package-specific rules

Several packages carry their own `.cursor/rules/` (e.g. `packages/cache`, `packages/storage`) documenting their Result-type APIs, branded cache keys, and design constraints. Read the relevant one before changing those packages.
