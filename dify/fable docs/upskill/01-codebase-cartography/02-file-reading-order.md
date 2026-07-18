# File Reading Order

Thirty files, in order. For each: why it matters, what to look for, what to ignore. Don't read linearly top-to-bottom inside big files — read the named ranges.

## Phase A — Orientation (junior path starts here)

| # | File | Why / look for | Ignore |
| --- | --- | --- | --- |
| 1 | `AGENTS.md` (root) | The repo's own architecture summary; layering rules | — |
| 2 | `api/AGENTS.md` | House style: session patterns, tenant scoping rule, controller/service split | The docstring policy details |
| 3 | `web/AGENTS.md` | Overlay/design-token contracts, i18n rule | — |
| 4 | `Makefile` | Every dev command: `dev-setup`, `format`, `lint`, `type-check`, `test` (L66-L105) | Docker push targets |
| 5 | Root `package.json` + `pnpm-workspace.yaml` | pnpm 11, Node ^22, catalog-pinned deps, workspace members | The full catalog list |
| 6 | [api/app_factory.py](../../../api/app_factory.py#L49-L219) | App factory, hook registration, ordered extension init | Enterprise license branch details |
| 7 | `api/dify_app.py` + `api/app.py` | What `DifyApp` adds over `Flask`; the actual entrypoint | — |
| 8 | `api/configs/` (skim package layout) | Pydantic settings; `dify_config` is the only env access path | Individual option docs |

## Phase B — One request end to end

| # | File | Why / look for | Ignore |
| --- | --- | --- | --- |
| 9 | [api/controllers/console/app/completion.py](../../../api/controllers/console/app/completion.py#L146-L200) | Canonical controller: decorator stack, Pydantic payload, error translation | The completion (non-chat) twin — same shape |
| 10 | [api/controllers/console/wraps.py](../../../api/controllers/console/wraps.py#L306-L337) | `edit_permission_required`, `is_admin_or_owner_required` — authorization as decorators | Billing/rate-limit decorators on first pass |
| 11 | [api/libs/login.py](../../../api/libs/login.py#L51-L118) | `login_required` + CSRF check (L99); `current_user` LocalProxy | Flask-Login docstring boilerplate |
| 12 | [api/services/app_generate_service.py](../../../api/services/app_generate_service.py#L87-L270) | Quota, rate limit, AppMode dispatch, streaming-vs-blocking split | single_iteration/loop variants |
| 13 | [api/models/model.py](../../../api/models/model.py#L392-L501) | `App` model: tenant_id, mode, status flags; property side-effect smell at L471-L484 | The 20+ TypedDicts above it |
| 14 | [api/models/account.py](../../../api/models/account.py#L87-L260) | `Account`, `Tenant`, role properties; sessions opened inside setters | — |

## Phase C — Frontend mirror of the same request (JS-stack home turf)

| # | File | Why / look for | Ignore |
| --- | --- | --- | --- |
| 15 | [web/service/fetch.ts](../../../web/service/fetch.ts#L122-L249) | ky client, CSRF header injection (L184), error toast hook | Marketplace special-casing |
| 16 | [web/service/base.ts](../../../web/service/base.ts#L452-L580) | `ssePost`: SSE consumption, 401→refresh→retry (L540-L545) | The 30-param `handleStream` signature — skim |
| 17 | [web/app/components/base/chat/chat/hooks.ts](../../../web/app/components/base/chat/chat/hooks.ts#L620-L709) | `useChat.handleSend`: optimistic tree updates, streaming callbacks | TTS/audio handling |
| 18 | [web/service/use-apps.ts](../../../web/service/use-apps.ts#L30-L81) | TanStack Query patterns: keys, invalidation on mutation | Statistics hooks |
| 19 | `web/app/(commonLayout)/` (layout.tsx + apps page) | App-router structure, providers | Styling |
| 20 | `web/app/components/workflow/store/workflow/` (slice list, `index.ts`) | Zustand slice composition for the graph editor | Individual slice bodies |

## Phase D — Persistence and async (mid path)

| # | File | Why / look for | Ignore |
| --- | --- | --- | --- |
| 21 | `api/models/dataset.py` (`Dataset` L166, `Document` L487, `DocumentSegment` L833, `ChildChunk` L1047) | RAG data model hierarchy | Pipeline template tables |
| 22 | `api/services/dataset_service.py#L1967-L2283` | Document creation → async dispatch (L2271-L2277) | The 2000 lines above |
| 23 | [api/tasks/document_indexing_task.py](../../../api/tasks/document_indexing_task.py) | Celery task anatomy, tenant-isolated queue (L192-L231), phased transactions | — |
| 24 | [api/core/indexing_runner.py](../../../api/core/indexing_runner.py#L69-L127) | extract→transform→load pipeline; per-doc error capture | `run_in_splitting_status` variants |
| 25 | `api/extensions/ext_database.py` + `ext_celery.py` + `ext_redis.py` | How db/celery/redis are wired | Metrics extensions |
| 26 | One migration in `api/migrations/versions/` | Alembic upgrade/downgrade shape | Reading many of them |

## Phase E — Boundaries and engine (senior path)

| # | File | Why / look for | Ignore |
| --- | --- | --- | --- |
| 27 | [api/controllers/service_api/wraps.py](../../../api/controllers/service_api/wraps.py#L55-L120) | Token auth boundary; tenant checks; end-user attachment | overloads at top |
| 28 | [api/services/account_service.py](../../../api/services/account_service.py#L200-L243) | JWT issuance + `authenticate`; refresh tokens in Redis (L150-L159) | Mail-token flows |
| 29 | [api/core/workflow/workflow_entry.py](../../../api/core/workflow/workflow_entry.py#L139-L234) | GraphEngine assembly: layers, limits, call-depth invariant (L172-L175) | Child-engine builder detail |
| 30 | `api/tests/` layout + one unit test near code you read | Test taxonomy: unit (local) vs integration (CI-only, root `AGENTS.md`) | Container tests |

## Paths

- **Junior**: A + B + C. Goal: locate and explain.
- **Mid**: A–D. Goal: could you add a field across all four layers safely?
- **Senior**: B, D, E, then straight to [../03-architecture-and-patterns/06-architecture-critique.md](../03-architecture-and-patterns/06-architecture-critique.md). Goal: what would you change, what would it cost?

Drill: after Phase B, close everything and draw the request path for `POST /console/api/apps/<id>/chat-messages` from memory — every file it touches until the first SSE byte. Check yourself against [05-key-flows.md](05-key-flows.md) Flow 1.
