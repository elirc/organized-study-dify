# Runtime and Tooling Map

## Toolchain (commands __inferred__ from Makefile / package.json unless noted)

| Task | Backend (`api/`) | Frontend (`web/`, run from root) |
| --- | --- | --- |
| Package manager | `uv` (`uv sync --dev`, `uv run --project api …`) | `pnpm@11.1.1`, Node `^22.22.1` |
| Dev server | `uv run --project api flask run --port 5001 --debug` | `pnpm dev` (concurrently: `vinext` dev + dev proxy — root `package.json`) |
| Worker | `uv run --project api celery -A app.celery worker …` | — |
| Format | `make format` (ruff format) | eslint `--fix` via `pnpm lint:fix` |
| Lint | `make lint` (ruff + import-linter + dotenv-linter) | `pnpm lint` (ESLint 10, antfu config + tsslint) |
| Types | `make type-check` (pyrefly + mypy) | `pnpm type-check` |
| Unit tests | `make test` or `make test TARGET_TESTS=./api/tests/unit_tests/<dir>` (pytest, xdist) | vitest (see `web/vitest.setup.ts`, `web/docs/test.md`) |
| E2E | — | Playwright + Cucumber in `e2e/` |
| Migrations | `uv run --project api flask db upgrade` (Alembic via Flask-Migrate; `ext_migrate`) | — |
| Full dev bootstrap | `make dev-setup` (middleware compose + envs + installs + migrate) | included |

Notable frontend toolchain quirk: `vite`/`vitest` are **overridden** to Void Zero's `vite-plus` packages, and `next dev` is replaced by `vinext` (root `package.json` scripts + `pnpm-workspace.yaml` overrides). Don't assume stock Next tooling behavior; check `vite.config.ts` and `web/next.config.ts`.

## Runtime boundaries

```
Browser (React 19 client components, TanStack Query cache, Zustand stores)
   │  HTTPS: JSON + SSE; cookies (console session) + CSRF header
   ▼
Next.js server / dev proxy  ── mostly a shell; API calls go to Flask
   ▼
Flask API (gunicorn in prod; Socket.IO wrapped WSGI — app_factory.py#L127-L138)
   │            │
   │            └── Redis: cache, rate limits, refresh tokens, pub/sub for SSE replay,
   │                        Celery broker, tenant-isolated task queues
   ├── PostgreSQL (or MySQL) via SQLAlchemy
   ├── Vector store (Weaviate default in dev compose) for embeddings
   ├── Storage (local/S3/… via extensions/storage)
   ├── ssrf_proxy container — all outbound user-controlled HTTP must go through
   │    core/helper/ssrf_proxy (api/AGENTS.md)
   ├── plugin_daemon container — runs plugins out-of-process
   └── Celery workers (same codebase, different entrypoint: celery_entrypoint.py)
        └── queues: dataset, priority_dataset (document_indexing_task.py#L234-L259), mail, …
```

Boundary rules worth internalizing:

- **Code in `api/core/` runs in both web requests and Celery workers.** It must not assume `flask.request` exists. That's why context is captured explicitly (`workflow_entry.py#L182-L183` `capture_current_context()`).
- **The browser never talks to Postgres/Redis**; the console UI's only contract is the console API. The shared web app pages use the web API with passport JWTs instead of cookies+CSRF (`web/service/base.ts#L502-L506`).
- **Streaming**: SSE responses may be produced by a *different process* than the one executing the workflow — execution happens in a Celery task publishing events through Redis; the HTTP request subscribes and replays (`app_generate_service.py#L41-L85`, `#L156-L185`). Transport is configurable: Redis Streams (durable, replayable) vs Pub/Sub (at-most-once) — the code adapts subscription gating accordingly.

## Environment variables

- Backend: **never read `os.environ` directly** — everything flows through `configs.dify_config` (Pydantic settings; rule in `api/AGENTS.md`). Templates: `api/.env.example`, `docker/envs/middleware.env.example` (neither file is in this filtered study export; see `09-reference/verification-log.md`). Examples used in code you've read: `PUBSUB_REDIS_CHANNEL_TYPE`, `APP_MAX_ACTIVE_REQUESTS`, `WORKFLOW_CALL_MAX_DEPTH`, `GRAPH_ENGINE_MIN_WORKERS`, `BATCH_UPLOAD_LIMIT`, `SECRET_KEY` (signs every JWT — `libs/passport.py`).
- Frontend: typed env via `@t3-oss/env-nextjs` (`web/env.ts`); public config constants in `web/config/`.

Interview angle: "How do you keep config out of code?" — mid-level answer: centralized typed settings object (`dify_config` / `env.ts`), example of a fail-fast validation, and the secret-handling rule (SECRET_KEY signs auth tokens, so rotation invalidates all sessions — a real operational tradeoff).

Drill: without running anything, write down what breaks if Redis is down but Postgres is up. Check yourself: login refresh tokens (`account_service.py#L150-L159`), rate limiting, SSE event replay, and *all* Celery dispatch — but classic blocking chat completions in dev might still partially work. Solid answers name at least three; strong answers separate hard failures from degraded modes.
