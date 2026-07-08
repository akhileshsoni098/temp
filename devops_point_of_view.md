# Southlake Insurance Platform — DevOps Handoff

Technical reference for deploying, operating, and maintaining this system. Companion to `appFlow.md` (non-technical, functional walkthrough) — this document covers architecture, environment, build/deploy, and known gaps.

---

## 1. Architecture at a glance

Two **independent** applications, two **independent** git repos, no monorepo tooling, no shared build:

| | Repo | Type | Talks to |
|---|---|---|---|
| Backend | `southlake_service` | NestJS 11 REST API | PostgreSQL, SMTP |
| Frontend | `southlake_ui` | Angular 22 SPA (client-side only, no SSR) | Backend API over HTTP |

They communicate purely over HTTP/CORS. Each is built, versioned, and deployed separately.

```
 Browser
    │
    ▼
 Angular SPA (static files: dist/southlake_ui/browser)
    │  HTTP calls to apiUrl (see §4 — currently broken in prod build, see §8)
    ▼
 NestJS API (all routes under /api/*, default port 3000)
    │
    ├──▶ PostgreSQL (primary datastore)
    ├──▶ SMTP (login OTP + invite emails)
    └──▶ Local disk ./uploads (MGA/State/Risk-Company documents)
```

There is **no Docker, no docker-compose, no CI/CD pipeline anywhere in this codebase today**. All of that needs to be authored fresh — this document exists to give you the facts needed to do that correctly rather than by trial and error.

---

## 2. Tech stack

**Backend (`southlake_service`)**
- Node.js `>=24` (also pinned in `.nvmrc`)
- NestJS `^11`, TypeORM `^0.3.20`, `pg` driver `^8.13`
- PostgreSQL `>= 14`
- Auth: **not JWT** despite naming in some docs — opaque random session tokens, hashed and stored in a `user_sessions` DB table (see §7 caveat)
- bcryptjs (password/OTP hashing), nodemailer (SMTP), helmet, class-validator, `@nestjs/throttler`

**Frontend (`southlake_ui`)**
- Node.js `>=24`
- Angular `^22`, ag-Grid Community `^36` (no enterprise license needed)
- Test runner: Vitest (not Karma)
- Pure client-side SPA — no SSR, no prerendering

---

## 3. Environment variables (backend)

Source of truth: `southlake_service/.env.example`. Real `.env` is correctly git-ignored — never commit it.

| Variable | Purpose |
|---|---|
| `NODE_ENV` | `development` / `production` — also toggles TypeORM query logging |
| `PORT` | API listen port (default `3000`) |
| `DATABASE_HOST/PORT/NAME/USER/PASSWORD` | PostgreSQL connection — **app fails fast at boot if HOST/NAME/USER are missing** |
| `DB_POOL_MAX` | Connection pool size (default 20) — **used in code but missing from `.env.example`, add it there** |
| `MAIL_HOST/PORT/USER/PASSWORD/FROM` | SMTP for login OTP + invite emails |
| `APP_URL` | Frontend base URL, used to build links inside outgoing emails |
| `CORS_ORIGIN` | Allowed browser origin — **if unset, defaults to allow-all. Must be set explicitly in production.** |
| `SESSION_EXPIRY_HOURS` | Login session validity window |
| `OTP_EXPIRY_MINUTES` / `OTP_MAX_ATTEMPTS` | OTP login code rules |
| `RATE_LIMIT_TTL` / `RATE_LIMIT_MAX` | Global throttler window/limit (applies to every route) |
| `STARLIGHT_DB_*` | Only needed for the one-time legacy data import script; not needed for normal operation |

**Known discrepancy — flag to the dev team, don't just "fix" silently:** the local `.env` and README reference `JWT_SECRET`, but nothing in the source code reads it (auth is DB-session-based, not signed JWT). It's dead configuration. Get an explicit decision on removing it vs. wiring it up before treating it as a real secret to rotate.

The backend loads `.env` via a relative path from `src/database/data-source.ts` (`../../.env`) — in a container, either bake a real `.env` into the image/mount it, or ensure this resolves correctly against the compiled `dist/` path.

---

## 4. Environment config (frontend) — **action required before prod deploy**

- `environment.ts` (dev): `apiUrl: 'http://localhost:3000/api'`
- `environment.prod.ts` (prod): `apiUrl: '/api'` (expects API reverse-proxied under the same origin)

**Bug: `angular.json`'s production build has no `fileReplacements` entry to swap these files.** Running `ng build --configuration production` today still bundles the *dev* `environment.ts` (`http://localhost:3000/api`) into the output. Every browser that loads the production build will try to call `localhost:3000` instead of the real API. **This must be fixed before any real deployment** — either add the standard Angular `fileReplacements` block in `angular.json`, or move to a runtime-config pattern (fetch `apiUrl` from a `config.json` served alongside `index.html`).

---

## 5. Database — migrations & seeding are manual, not automatic

Nothing runs migrations or seeds on app boot. Deployment order for a fresh environment:

```bash
# Backend, in order:
npm ci
# create/populate .env
npm run migration:run   # applies all pending migrations
npm run seed             # first time only — roles, superadmin user, permissions, COA, master data
npm run build
npm run start:prod       # node dist/main
```

- `npm run migration:run` / `migration:revert` / `migration:generate` / `migration:create` / `migration:check` (drift check — non-zero exit if entities don't match migrations, wire this into CI as a guard rail)
- `npm run seed` is idempotent — safe to re-run
- `npm run db:reset` / `db:refresh` are **destructive** (drop + recreate DB) — dev/local only, never point at a prod DB

`synchronize: false` is correctly set everywhere — schema changes only ever happen via explicit migrations. Good practice already in place; keep it that way.

---

## 6. File uploads — needs a persistent volume

MGA, State, Risk-Company, and Chart-of-Accounts document uploads use `multer` with `diskStorage`, hardcoded to `./uploads` relative to the process working directory. There is **no S3/cloud storage integration and no configurable upload path (no env var)**.

**Implication for containerized/multi-replica deployment:**
- Mount a persistent volume at `<container-workdir>/uploads`, or
- Migrate to object storage (S3/Azure Blob/etc.) before scaling beyond a single instance — otherwise uploaded documents are lost on restart and inconsistent across replicas.

Separately, `batch-upload` and the ITD Excel seeder use in-memory multer storage (not persisted to disk) — those are fine as-is, just parsed and discarded. Max upload size is hardcoded to 25 MB (`workbook.constants.ts`).

---

## 7. Auth model, security posture

- Global prefix: all API routes under `/api/*`
- Swagger UI: **`/api/docs`** (note: backend README says `/api`, that's stale — trust this document)
- CORS: `helmet()` + explicit CORS middleware, credentials enabled
- Global `ValidationPipe`: `whitelist: true` (unknown body fields silently stripped — be aware of this if debugging "field didn't save" issues), `transform: true`
- Global rate limiting via `@nestjs/throttler`, applied to every route by default (two profiles: `default` and `short`)
- **Every write endpoint now enforces server-side RBAC** via `@RequirePermission(...)` decorators checked against the caller's role + per-user permission overrides (this was audited and hardened — see note in §9)
- Graceful shutdown enabled (`enableShutdownHooks()`) — plays well with rolling deploys/K8s `SIGTERM`

**No health-check endpoint exists.** No `@nestjs/terminus`, no `/health`/`/status` route. Add one before putting this behind a load balancer or K8s readiness/liveness probe — otherwise you'll need to fake it with `GET /api/docs` as a crude liveness check.

**No structured logging, no APM/monitoring.** Only NestJS's built-in `Logger` → stdout/stderr. Fine for basic container log collection, but there's no Sentry/Datadog/OpenTelemetry/Winston/Pino — add if you need alerting or distributed tracing.

---

## 8. What's missing / what to build before production

In priority order:

1. **Fix the frontend prod `apiUrl` bug** (§4) — otherwise the app is non-functional in any real deployment.
2. **Dockerfile for both apps** — none exist. Backend: standard Node runtime image running `dist/main`. Frontend: multi-stage build → static files served via nginx/Caddy with SPA fallback routing (serve `index.html` for unknown paths) and a reverse proxy for `/api` → backend.
3. **`CORS_ORIGIN`** must be set to the real frontend origin in every non-local environment — the current default (allow-all) is a production risk if left unset.
4. **Health-check endpoint** for load balancer/orchestrator probes.
5. **Persistent volume or S3 migration** for `./uploads` (§6).
6. **CI/CD pipeline** — none exists (no `.github/workflows`, no `.gitlab-ci.yml`). Suggested minimum: lint → typecheck → `migration:check` (drift guard) → test → build → deploy.
7. Decide the fate of `JWT_SECRET` (§3) — remove or implement, don't leave it as a false sense of security.
8. Add `DB_POOL_MAX` to `.env.example` for visibility.
9. Consider structured logging + basic APM once there's real traffic to debug.

---

## 9. Recent hardening (context for whoever inherits this)

The RBAC (role/permission) system was recently audited end-to-end and several real gaps were closed — worth knowing so you don't mistake the current strict permission checks for bugs:

- Every Masters/Accounting/Reporting controller now requires the correct `@RequirePermission('module.action')` server-side (previously several had none — UI-only gating, bypassable via direct API calls).
- Several controllers (Database Seeder, all Workbook/Reports endpoints) previously had a blanket `@Public()` decorator, meaning **no login was required at all** — this has been removed; they now require authentication + the correct permission like everything else.
- Default seed data intentionally does **not** grant "delete" permissions to the non-superadmin "admin" role (mirrors how `user.delete` already worked) — this is a deliberate secure-by-default choice, not an oversight. A superadmin can grant delete access to any role via the Roles & Permissions UI at any time.

If something that used to "just work" now returns `403 Forbidden`, check whether the calling role/user actually has the corresponding `module.action` permission granted — this is very likely intentional, not a regression.
