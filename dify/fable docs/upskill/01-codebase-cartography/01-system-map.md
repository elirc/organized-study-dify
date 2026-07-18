# System Map

## Repo shape: polyglot monorepo

```
dify/
├── api/            Python 3 / Flask backend (layered: controllers → services → core)
│   ├── controllers/  HTTP surface: console | service_api | web | files | mcp | inner_api | trigger
│   ├── services/     Application services (use-cases, orchestration)
│   ├── core/         Domain engine: app runners, workflow, rag, tools, plugin, llm
│   ├── models/       SQLAlchemy models (multi-tenant tables)
│   ├── tasks/        Celery tasks (queues: dataset, priority_dataset, mail, …)
│   ├── extensions/   Flask extension initializers (db, redis, celery, login, storage…)
│   ├── migrations/   171 Alembic migration files
│   ├── libs/         Cross-cutting helpers (login, passport/JWT, datetime…)
│   ├── configs/      Pydantic settings — the only legal way to read env vars
│   └── tests/        unit_tests/ (local), integration_tests/ (CI-only)
├── web/            Next.js 16 + React 19 + TS frontend (console + shared web apps)
│   ├── app/          App-router routes + components (the big one)
│   ├── service/      HTTP layer: ky client, SSE reader, TanStack Query hooks
│   ├── models/ types/ context/ hooks/ i18n/  supporting layers
├── packages/       Workspace packages: dify-ui (design system), contracts, tsconfig…
├── docker/         Compose files: full stack + docker-compose.middleware.yaml (dev deps)
├── dify-agent/     Separate Python project (agent backend) — out of curriculum scope
├── e2e/            Playwright/Cucumber end-to-end tests (workspace member)
└── sdks/           Client SDKs (e.g. nodejs-client)
```

Key structural facts (verified):

- The Flask app is assembled by an **application factory**: [api/app_factory.py](../../../api/app_factory.py#L49-L138) creates the app, applies a global `before_request` (enterprise license gate) and `after_request` (trace headers), then wraps it in a Socket.IO WSGI app (L127-L138). Twenty-seven extension modules initialize in an explicit, ordered list (L177-L206) — order matters (comment at L192: logstore after storage, before celery).
- The **workflow graph engine is not in this repo**: `graphon~=0.3.1` is an external dependency (`api/pyproject.toml:46`). This repo builds graphs and layers around it ([api/core/workflow/workflow_entry.py](../../../api/core/workflow/workflow_entry.py#L185-L220)).
- Frontend workspace members are declared in `pnpm-workspace.yaml`: `web`, `e2e`, `sdks/nodejs-client`, `packages/*`, with a large version `catalog:` pinning every dependency.

## Ownership map

| Concern | Owner | Must NOT own |
| --- | --- | --- |
| HTTP parsing, status codes, auth decorators | `api/controllers/` | Business logic (per `api/AGENTS.md`: "Controllers: parse input via Pydantic, invoke services… no business logic") |
| Use-case orchestration, task dispatch, permissions | `api/services/` | HTTP details, raw response shaping |
| Domain engine: app generation, workflow, RAG, LLM invocation | `api/core/` | Flask request context assumptions (it also runs inside Celery workers) |
| Table shape + row-level helpers | `api/models/` | Heavy business rules (see the `App.is_agent` violation, `api/models/model.py#L471-L484`) |
| Background execution | `api/tasks/` + `api/extensions/ext_celery.py` | Anything that assumes a logged-in web session |
| UI routes/components | `web/app/` | Direct fetch calls (must go through `web/service/`) |
| HTTP/SSE client, query caching | `web/service/` | Component state |
| Design-system primitives (overlays, tokens) | `packages/dify-ui` | Product logic (`web/AGENTS.md` mandates overlay imports from `@langgenius/dify-ui/*` only) |

## Public interfaces vs private internals

**Public (contracts — changing them has blast radius beyond this repo):**

- **Service API** (`api/controllers/service_api/`) — the versioned customer-facing REST API, authenticated by app API tokens ([wraps.py](../../../api/controllers/service_api/wraps.py#L55-L120)).
- **Web app API** (`api/controllers/web/`) — powers published/shared chatbot pages, JWT "passport" auth.
- **Console API** (`api/controllers/console/`) — powers the dashboard UI. Technically internal, but the web app pins to it, so treat it as a contract between the two halves of the monorepo.
- **MCP + trigger + files controllers**, **OpenAPI schemas** (`api/openapi/`), **SDKs** (`sdks/`).
- **DSL import/export** (`services/app_dsl_service.py`) — app definitions as YAML shared between installations.

**Private (refactor freely if tests pass):** `api/core/**` internals, `api/services/**` signatures, web components, Zustand store slices.

Senior noticing: the console API and service API expose *the same domain* through different auth models (session+CSRF vs bearer token). When you add a feature, you often must decide whether it belongs to one, the other, or both — that decision is an API-design interview question hiding in plain sight.

## Drill

Without opening more files, predict: which directory handles "a scheduled trigger fires a workflow at 9am"? Then verify by listing `api/controllers/trigger/`, `api/core/trigger/`, `api/services/trigger/` and `api/schedule/`. Self-grade — Basic: found the folders. Solid: can say which layer dispatches Celery work. Strong: can explain why trigger execution must not depend on Flask request context (workers have none), citing how `core/` stays context-free.
