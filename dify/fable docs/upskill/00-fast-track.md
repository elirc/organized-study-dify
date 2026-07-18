# Fast Track — One Weekend

Goal: by Sunday night you can run Dify locally, trace two end-to-end flows, make one safe change, run one test, and explain a flow aloud as if in an interview.

## Saturday morning — install and run

All commands below are __inferred__ from `Makefile`, root `package.json`, and `AGENTS.md` (not executed during authoring — see [09-reference/verification-log.md](09-reference/verification-log.md)). Prerequisites: Docker, Node ^22.22.1 + pnpm 11 (root `package.json#L6-L8`), Python + [uv](https://docs.astral.sh/uv/).

```bash
# 1. Middleware (Postgres, Redis, Weaviate, sandbox, plugin daemon) + both envs
make dev-setup            # Makefile — copies env files, starts docker compose,
                          # pnpm install, uv sync, flask db upgrade

# 2. API (terminal 1) — from api/
uv run --project api flask run --host 0.0.0.0 --port 5001 --debug

# 3. Celery worker (terminal 2) — background tasks won't run without this
uv run --project api celery -A app.celery worker -Q dataset,generation,mail,ops_trace

# 4. Web (terminal 3) — from repo root
pnpm dev
```

Open the console UI, complete setup (`/install`), create a workspace. If something fails, check `docker/docker-compose.middleware.yaml` service names (`db_postgres` L3, `redis` L83, `weaviate` L226) against your `.env`.

## Saturday afternoon — trace flow #1: sending a chat message

Read these in order; predict before you open each next file.

1. [web/app/components/base/chat/chat/hooks.ts](../../web/app/components/base/chat/chat/hooks.ts#L620-L709) — `handleSend`: builds optimistic placeholder question/answer nodes in a chat *tree* (L644-L668), assembles `bodyParams` with `response_mode: 'streaming'` (L685-L692), then fires `ssePost` (L1162-L1168).
2. [web/service/base.ts](../../web/service/base.ts#L452-L580) — `ssePost`: raw `fetch` with CSRF header (L503), retries once after refreshing the access token on 401 (L540-L545), pipes the body through `handleStream` (L555+), which dispatches ~30 typed SSE event callbacks.
3. [api/controllers/console/app/completion.py](../../api/controllers/console/app/completion.py#L146-L200) — `ChatMessageApi.post`: the decorator stack is the security story (`@setup_required @login_required @account_initialization_required @get_app_model @edit_permission_required`), Pydantic validates the payload (L161), then it delegates to the service and translates domain errors to HTTP errors (L179-L200).
4. [api/services/app_generate_service.py](../../api/services/app_generate_service.py#L87-L270) — `generate`: reserves billing quota (L107-L112), enters an app-level rate limit (L114-L119), dispatches by `AppMode` (L124+). Note the `except: refund + exit` at L264-L267.
5. Skim [api/core/workflow/workflow_entry.py](../../api/core/workflow/workflow_entry.py#L139-L234) — where advanced-chat/workflow apps hand off to the external `graphon` graph engine with layered limits.

Pause-and-predict: what happens if the browser disconnects mid-stream? (Hint: for ADVANCED_CHAT streaming, execution runs in a **Celery task**, not the request thread — `app_generate_service.py#L156-L185` — and events are replayed from Redis.)

## Sunday morning — trace flow #2: uploading a knowledge document

1. `api/services/dataset_service.py#L1967-L2060` — `save_document_with_dataset_id`: billing/limit checks, then document rows are created and committed.
2. `api/services/dataset_service.py#L2271-L2277` — the async dispatch: `DocumentIndexingTaskProxy(...).delay()`.
3. [api/tasks/document_indexing_task.py](../../api/tasks/document_indexing_task.py#L49-L190) — the Celery task: re-checks limits *inside* the task (L64-L91), sets status `PARSING` in a short transaction (L93-L106), then runs the indexer with **no ambient transaction** (L108-L120).
4. [api/core/indexing_runner.py](../../api/core/indexing_runner.py#L69-L127) — extract → transform → load segments → load into the vector index; per-document error capture writes `indexing_status = ERROR` back to the row (L58-L67).

## Sunday afternoon — one safe change + one test

- **Change**: pick any UI string; find it under `web/i18n/en-US/` (hardcoded text is forbidden — `web/AGENTS.md`), tweak it, watch hot reload.
- **Test** (__inferred__ commands):

```bash
# Backend: one unit test file
make test TARGET_TESTS=./api/tests/unit_tests/libs
# Frontend: vitest for one spec
pnpm --filter dify-web test service/base.spec.ts   # see web/package.json for exact script name
```

- **Teach-back exercise**: stand up and explain flow #1 aloud in 3 minutes as if an interviewer asked "walk me through how a chat message gets answered in a system you know." Must include: where validation happens, where authorization happens, why streaming uses SSE + Celery instead of holding the request thread, and one failure mode (e.g., client never subscribes → the subscribe-gated task start with fallback timer, `app_generate_service.py#L41-L85`).

## What the fast path skips

The workflow graph editor internals (ReactFlow + Zustand slices), the plugin system (`api/core/plugin/`), model provider management, RAG retrieval/rerank internals, enterprise/billing branches, and all of `dify-agent/`. The full curriculum covers the first four; enterprise code is intentionally out of scope.
