# Web App / SaaS Heuristics

For server-rendered or API-driven web applications: SaaS products, internal tools, customer-facing dashboards, B2B platforms.

## When this applies

Concrete signals (need ≥ 2):

- An HTTP server bootstrap in `entry_points_candidates` (Fastify / Express / Koa / Next.js / Django / Flask / Rails / Spring Boot / Phoenix / Gin / Echo).
- Persistent storage: SQLite / Postgres / MySQL migrations directory, ORM models, Prisma/SQLAlchemy/ActiveRecord schemas.
- Route registration code matching the framework's idiom.
- An auth system (sessions, JWT, OAuth) in deps.
- A UI codebase (React / Vue / Svelte / templates) — may be the same repo or peer.

## Typical top-level modules

| Typical module | One-line responsibility |
|---|---|
| `app-wiring` | Server bootstrap, DI graph, middleware order, error handler — where everything is composed |
| `config` | Validated env / typed configuration — single source of truth for runtime parameters |
| `db` (or `persistence`) | Connection pool, migration runner, base repository |
| `migrations` | Schema evolution log (separate from db engine wrapper) |
| `auth` (user) | Login / register / session management for end users |
| `admin-auth` | Separate admin account + session system (usually distinct from user auth) |
| `<business-domain-N>` | One module per business concept (e.g., `billing`, `orders`, `inventory`, `posts`, `assets`) — these are the majority of the project |
| `integrations` | Adapters for external services (Stripe / SendGrid / S3 / OAuth providers) |
| `jobs` (or `workers`) | Background workers, queues, scheduled jobs |
| `ui` (or `frontend`) | Browser UI codebase (may be its own top-level module in monorepo) |
| `observability` | Logging, metrics, tracing — pino / OpenTelemetry / Sentry |
| `shared` (or `common`) | Cross-cutting utilities: clock, error types, ID generators, validation helpers |
| `cli` / `scripts` | One-off ops scripts (db seed, data migration, debugging tools) |
| `tests` / `fixtures` | Shared test infrastructure if not co-located with each domain module |

## Common boundary pitfalls

| Pitfall | Why it's wrong |
|---|---|
| Splitting by file type (controllers / services / repositories) | This is layering, not modules. A change to "billing" must touch all three layers — but a `billing` change is one unit of work. Module ≠ layer. |
| Putting all routes under one `api` module | Routes belong to their business domain. `api` as a top-level module is a sign of unsplit business logic. |
| Lumping `auth` and `admin-auth` | They have different sessions, different password requirements, different audit logs. Two separate concerns. |
| Treating `migrations` as part of `db` | Migrations have a different lifecycle (additive-only contract, ordered, irreversible) than runtime db code. Separate. |
| Lumping `integrations` into one module | Each external service has its own failure mode, its own credential, its own version. Sub-modules per provider. |
| Putting `jobs` under the business domain they serve | Jobs cross domains (e.g., a daily report job touches billing + analytics). Cron / worker registration is a distinct concern. |
| Forgetting `config` as a separate module | Env handling is often centralized in one file (e.g., `config.ts` with Zod schema). Treating it as part of `app-wiring` hides the source-of-truth nature. |
| Treating `shared` as a single leaf | If `shared` contains error types, clock, ID, RRF, etc. — these may be separable. Be careful here: sometimes one cohesive utility set, sometimes a god-bag. Apply leaf criteria honestly. |

## Naming conventions

- Business modules: domain noun (`billing`, `inventory`, `posts`), NOT verb (`process-billing`).
- Auth split: `auth` (users) and `admin-auth` (admins) — match the project's session-table split.
- Workers: project term wins (`jobs` / `workers` / `tasks` / `crons`).
- Routes: never name a module after a URL path. Modules own business concepts, not URL shapes.

## Risk hotspots — almost always need `RISKS.md` entries

- Database migrations (especially destructive ones — `DROP COLUMN`, `ALTER TYPE`).
- Auth password storage (algorithm, cost factor, pepper).
- Session token lifetime / rotation / revocation.
- Authorization checks per route (missing check = privilege escalation).
- Rate limiting (or its absence) on login / signup / public APIs.
- Persistent state mutations from background jobs (race with HTTP handlers).
- Money / billing accuracy (rounding, currency conversion, idempotency).
- PII / credential storage in logs (accidental leakage).
- File uploads (size limit, type validation, path traversal).
- N+1 query problems in hot paths.

## Notes specific to this domain

- **One business module = one folder pattern**. Most well-organized web apps follow `modules/<domain>/{types,repository,service,routes,index}.ts` or equivalent. If the project follows this pattern, each `modules/<domain>/` is a strong candidate for one MTM module. **Do not split internally** unless leaf criteria say so.
- **The `app-wiring` module is almost always a single leaf** — it has one cohesive responsibility (composition) even if it's 800 lines. See `M0010-hub-app-wiring` style.
- Frontend UI may or may not deserve its own top-level module — depends on whether it's monorepo or peer repo. In monorepo, yes.
- Cron registration usually lives in `app-wiring`, but the cron *implementation* belongs to the business domain it serves. Edges in `interactions/runtime.md` show this.
