# Key Flows

Six end-to-end traces. Each later module reuses a *different* flow for its examples — learn them all. Anchors verified 2026-07-10.

---

## Flow 1: Console chat message → streamed answer (UI/client + server)

Why this flow matters: it crosses every layer — React optimistic UI, SSE, auth decorators, service orchestration, rate limiting, and the LLM engine. It is the demo answer for half of your interview questions.

Open these files first:
- `web/app/components/base/chat/chat/hooks.ts#L620-L709` — `handleSend`
- `web/service/base.ts#L452-L580` — `ssePost`
- `api/controllers/console/app/completion.py#L146-L200` — `ChatMessageApi`
- `api/services/app_generate_service.py#L87-L270` — `generate`

Trace:

| Step | Owner | File | What happens | Data shape | Risk |
| --- | --- | --- | --- | --- | --- |
| 1 | React hook | `chat/hooks.ts#L620-L668` | Optimistic placeholder question+answer added to chat **tree** (`parent_message_id` = branching for regenerate) | `ChatItemInTree` | Placeholder desync if request fails silently |
| 2 | HTTP layer | `chat/hooks.ts#L685-L692`, `base.ts#L452-L521` | POST with `response_mode: 'streaming'`; CSRF header from cookie (`base.ts#L503`) | JSON body | 401 triggers token refresh + **recursive retry** (`#L540-L545`) |
| 3 | Controller | `completion.py#L146-L160` | Decorator stack: `setup_required → login_required → account_initialization_required → get_app_model(mode=[CHAT, AGENT_CHAT]) → edit_permission_required` | Flask view args | Order matters: authn before authz before resource load |
| 4 | Controller | `completion.py#L161-L166` | Pydantic `ChatMessagePayload` validates; UUID check on conversation ids (`#L60-L65`) | validated dict | `model_config` alias trick at `#L45` |
| 5 | Service | `app_generate_service.py#L107-L119` | Billing quota reserved; app-level `RateLimit.enter` | request_id | Refund+exit on exception (`#L264-L267`) |
| 6 | Service | `app_generate_service.py#L124-L151` | Mode dispatch → `ChatAppGenerator().generate(...)` wrapped as event stream | generator of SSE events | agent-chat coercion via `is_agent` (`#L121-L123`) |
| 7 | Core | `api/core/app/apps/chat/` | Prompt assembly, model invocation, message persistence (task pipeline) | LLM chunks | Provider errors surface as domain exceptions |
| 8 | Controller | `completion.py#L179-L200` | Domain exceptions → HTTP errors (404, quota, provider…) | error JSON | Bare `Exception` → 500 with log |
| 9 | React | `base.ts#L555-L566` + hook callbacks | `handleStream` dispatches `onData/onThought/onMessageEnd/...`; UI mutates the answer node in place | SSE lines | Swallowed abort errors are filtered by string match (`#L561`) |

Validation and authorization: Pydantic at step 4; session+CSRF at `libs/login.py#L86-L100`; role check (editor+) at `console/wraps.py#L306-L321`; app-mode check inside `get_app_model`.

Persistence and side effects: conversation+message rows (`models/model.py` `Conversation` L1044, `Message` L1393); quota counters in Redis; token usage records.

Tests that cover it: frontend `web/app/components/base/chat/chat/__tests__/hooks.spec.tsx` mocks `ssePost` and asserts callback handling; backend controller tests under `api/tests/unit_tests/controllers/`. No single test exercises the full path (normal for a flow this wide — E2E lives in `e2e/`).

What juniors usually miss: the chat is a **tree**, not a list; `edit_permission_required` means viewers can't even *debug-chat*; errors are translated per-type, not blanket-caught.

What seniors notice: the recursive 401 retry in `ssePost` has no retry-count guard (possible loop if refresh "succeeds" but the new token is also rejected — investigate); rate-limit exit only happens in `finally` for *non-streaming* (`#L268-L270`) because streaming exit is owned by the generator wrapper.

Interview angle: "Design a streaming chat backend" and "how do you handle optimistic UI with server-side failure?" — this flow is your evidence for both.

Drill: with the app running, send a message with devtools open. Identify each SSE event type you see and the file/line where the frontend handles it. Self-grade — Basic: name 3 event types. Solid: map each to its callback in `IOtherOptions` (`base.ts#L84-L132`). Strong: explain which events would differ if the app were mode `workflow`.

---

## Flow 2: Advanced-chat / workflow execution via Celery + event replay (background/async)

Why this flow matters: it shows how to decouple *execution* from *delivery* — the pattern behind every "how would you scale long-running requests?" interview answer.

Open these files first:
- `api/services/app_generate_service.py#L152-L208` — ADVANCED_CHAT branch
- `api/services/app_generate_service.py#L41-L85` — subscribe-gated task start
- `api/core/workflow/workflow_entry.py#L139-L234` — engine assembly
- `api/tasks/app_generate/workflow_execute_task.py` — the Celery consumer

Trace:

| Step | Owner | File | What happens | Data shape | Risk |
| --- | --- | --- | --- | --- | --- |
| 1 | Service | `app_generate_service.py#L153-L169` | Pick draft vs published workflow by `InvokeFrom` (`#L386-L420`); build `AppExecutionParams` JSON payload with fresh `workflow_run_id` | Pydantic → JSON | Draft used only for DEBUGGER |
| 2 | Service | `#L171-L185` | HTTP response = subscription to events for that run id; the Celery task is enqueued **on first subscribe** (or by a 200ms fallback timer for pub/sub transport) | SSE generator | If transport = pub/sub, events before subscribe are lost — hence gating |
| 3 | Worker | `tasks/app_generate/workflow_execute_task.py` | Deserializes payload, runs the generator app logic in a worker | — | Worker crash = run stuck; see pause/state layers |
| 4 | Engine | `workflow_entry.py#L172-L216` | Call-depth invariant (max nested workflows), `GraphEngine` + `ExecutionLimitsLayer` (max steps/time), `LLMQuotaLayer` | `GraphEngineEvent` stream | Runaway graphs bounded by config |
| 5 | Delivery | Redis streams/pubsub | Events published per `workflow_run_id`; HTTP side replays via `retrieve_events` | typed events | Streams = durable replay; pub/sub = at-most-once |
| 6 | Frontend | `chat/hooks.ts#L1140-L1155` | On `workflow_paused`, client re-subscribes via `sseGet /workflow/<run_id>/events` | SSE | Human-input pause/resume support |

Validation and authorization: same controller stack as Flow 1; additionally workflow existence/publication checks (`#L397-L418`).

Persistence and side effects: `WorkflowRun` rows, node execution records, pause-state persistence (`PauseStateLayerConfig`, `#L189-L204`).

Tests that cover it: unit tests under `api/tests/unit_tests/core/workflow/` (engine call-sites); the graphon engine itself is tested upstream.

What juniors usually miss: `streaming=False` runs the workflow **synchronously in the web request** (`#L186-L208`) — two totally different execution topologies behind one endpoint parameter.

What seniors notice: `workflow_run_id` is minted *before* enqueue, making the SSE subscription idempotent and reconnectable — that's the invariant enabling replay; the fallback timer (`SSE_TASK_START_FALLBACK_MS = 200`, `#L33`) trades a tiny latency window against tasks that never start when a client dies pre-subscribe.

Interview angle: "How do you deliver progress of a long job to the browser?" — answer with: enqueue + durable event log keyed by run id + replayable subscription, citing this flow, then contrast with WebSockets and polling.

Drill: write (on paper) the failure matrix: client disconnects {before subscribe, mid-stream, after completion} × transport {streams, pubsub}. What does the user see on reconnect in each cell? Strong = you cite where the code decides each behavior.

---

## Flow 3: Knowledge document upload → indexing (persistence + async)

Why this flow matters: classic write-path + background-pipeline design: quota checks, phased transactions, tenant-fair queueing, idempotent-ish steps, status machines.

Open these files first:
- `api/services/dataset_service.py#L1967-L2060` — `save_document_with_dataset_id`
- `api/services/dataset_service.py#L2271-L2277` — async dispatch
- `api/tasks/document_indexing_task.py#L49-L231`
- `api/core/indexing_runner.py#L69-L127`

Trace:

| Step | Owner | File | What happens | Data shape | Risk |
| --- | --- | --- | --- | --- | --- |
| 1 | Service | `dataset_service.py#L1975-L2006` | doc_form check; billing: batch limit, plan limits, vector-space quota | `KnowledgeConfig` | Checks duplicated in task (defense in depth) |
| 2 | Service | `#L2008-L2047` | First upload fixes dataset invariants: `indexing_technique`, embedding model, collection binding, default retrieval model | Dataset row | These are near-immutable afterwards — a real schema-design lesson |
| 3 | Service | `#L2049-L2269` | Document rows built per source (file/notion/website), batch id minted, committed | `Document` rows, `batch` | Long function; lock via `LockNotOwnedError` handling `#L2280` |
| 4 | Service | `#L2271-L2277` | `DocumentIndexingTaskProxy(...).delay()` → routes to normal vs priority queue | ids only | Payload = ids, not objects (serialization discipline) |
| 5 | Worker | `document_indexing_task.py#L93-L106` | Phase 1: status → `PARSING` in a short transaction, then **commit and close** | status enum | Avoids holding a tx across the long pipeline |
| 6 | Worker | `#L108-L120` | Phase 2: `IndexingRunner().run(documents)` with no ambient transaction | — | Errors captured per document |
| 7 | Core | `indexing_runner.py#L69-L119` | extract (parse file) → transform (split/clean/embed prep) → `_load_segments` → `_load` (vector store write) | `DocumentSegment` rows + vectors | Status machine: PARSING→…→COMPLETED/ERROR |
| 8 | Worker | `document_indexing_task.py#L192-L231` | Tenant-isolated queue: on finish, pull next queued docs for this tenant and re-enqueue; else clear flag | Redis list | Fairness: one tenant can't starve the `dataset` queue |
| 9 | Worker | `#L122-L189` | Optional summary-index task chained after success (high_quality only) | — | Chained task failure is logged, not fatal |

Validation and authorization: controller layer (console `datasets` controllers) enforces dataset permissions; service re-checks plan quotas; the task re-checks again (`#L64-L91`) because *the queue is a trust boundary* — anything could have changed between enqueue and execution.

Persistence and side effects: `Document`/`DocumentSegment`/`ChildChunk` rows; vector-store collection writes; Redis queue keys; storage reads.

Tests that cover it: `api/tests/unit_tests/tasks/` and `unit_tests/services/` include dataset/indexing coverage; end-to-end indexing needs real vector stores → CI-only integration tests (root `AGENTS.md`).

What juniors usually miss: the status write happens in its own committed transaction *before* the slow work, so the UI can show progress; error text is persisted onto the row (`indexing_runner.py#L58-L67`) — observability via the data model.

What seniors notice: step 5/6 split is a deliberate "short transaction, long work outside" pattern; the tenant queue's flag-set/clear protocol (`#L206-L231`) is a hand-rolled semaphore — worth probing for race conditions (what if the worker dies between pull and re-enqueue? tasks wait until the *next* completion clears/reprimes — investigate).

Interview angle: "Design a document ingestion pipeline" — you can now answer with quota gates, phased status transactions, per-tenant fairness, chained enrichment tasks, and DB-visible error states.

Drill: draw the `indexing_status` state machine by grepping `IndexingStatus` usages. Strong = you can say which component owns each transition and which transitions are terminal.

---

## Flow 4: Console login → JWT + refresh token (auth boundary)

Why this flow matters: it's the canonical authn story: credential check, token issuance, refresh rotation, CSRF defense.

Open these files first:
- `api/controllers/console/auth/login.py` — endpoint
- `api/services/account_service.py#L200-L243` — `get_account_jwt_token`, `authenticate`
- `api/libs/passport.py` — JWT sign/verify
- `api/libs/login.py#L51-L102` — request-side enforcement

Trace:

| Step | Owner | File | What happens | Data shape | Risk |
| --- | --- | --- | --- | --- | --- |
| 1 | Controller | `console/auth/login.py` | Parse email/password, call service, set cookies | — | Rate limiting on attempts (see service) |
| 2 | Service | `account_service.py#L214-L243` | Load account by email; banned check; salted-hash compare; PENDING→ACTIVE on first login | `Account` | Uniform "Invalid email or password" avoids user enumeration |
| 3 | Service | `#L200-L212` | JWT payload `{user_id, exp, iss: EDITION, sub: "Console API Passport"}` signed HS256 with `SECRET_KEY` | JWT string | Expiry from `ACCESS_TOKEN_EXPIRE_MINUTES` |
| 4 | Service | `#L142-L159` | Refresh token stored in Redis with TTL, both directions (token→account, account→token) | Redis keys | Logout = delete both; rotation invalidates old |
| 5 | Request-side | `libs/login.py#L86-L100` | `login_required`: resolve user, else 401 via login manager; then **CSRF check** (`check_csrf_token`, L99) | — | CSRF token also cookie-delivered; header echoed by `web/service/fetch.ts#L184` |
| 6 | Frontend | `web/service/refresh-token.ts` + `base.ts#L540-L545` | On 401, refresh then retry the original call | — | Concurrent-refresh stampede is the classic bug here — inspect how it dedupes |

Validation and authorization: authn here; authz is separate (role decorators, Flow 1 step 3). Keep those words distinct in interviews: *authentication = who you are; authorization = what you may do.*

Persistence and side effects: `last_login_at/ip` updates; Redis token keys.

Tests that cover it: `api/tests/unit_tests/services/` account service tests; `libs` tests for token helpers.

What juniors usually miss: two token systems coexist — console JWT (+refresh in Redis) and web-app "passport" JWTs for shared pages (`base.ts#L505`, `libs/passport.py`); also everything is signed with the *single* `SECRET_KEY`.

What seniors notice: `authenticate` also *sets* the password when arriving with a valid invite token and a passwordless account (`#L225-L232`) — a subtle multi-purpose method; commit happens once at the end (`#L241`).

Interview angle: "JWT vs session?" Mid answer: this repo uses JWT access + Redis-backed refresh (revocable), CSRF because cookies carry the session — explain why stateless-JWT-only makes logout hard.

Drill: list every place a request can be rejected before the view body runs, in order, for `POST /console/api/apps/<id>/chat-messages`. (Answer spans `before_request` license hook `app_factory.py#L59-L96`, then the five decorators.) Strong = correct order with file/line.

---

## Flow 5: Service API request with app token (security boundary + multi-tenancy)

Why this flow matters: it's the *other* front door — machine-to-machine auth — and the cleanest place to study tenant isolation.

Open these files first:
- `api/controllers/service_api/wraps.py#L55-L120` — `validate_app_token`
- `api/controllers/service_api/app/completion.py` — a consumer endpoint

Trace:

| Step | Owner | File | What happens | Data shape | Risk |
| --- | --- | --- | --- | --- | --- |
| 1 | Decorator | `service_api/wraps.py#L61` | Bearer token → `ApiToken` lookup (cached, single-flight — `services/api_token_service.py`) | `ApiToken` | Token cache staleness vs revocation |
| 2 | Decorator | `#L63-L71` | App exists, `status == "normal"`, `enable_api` true | `App` | Disabled API returns 403 not 404 |
| 3 | Decorator | `#L73-L77` | Tenant exists and not archived | `Tenant` | Tenant lifecycle enforced at the edge |
| 4 | Decorator | `#L82-L103` | Optional end-user resolution from query/json/form → `EndUser` set as `current_user` | `EndUser` | Caller-supplied `user` string = identity! (see below) |
| 5 | Decorator | `#L104-L120` | No end user? Log in the **tenant owner's Account** so `current_account_with_tenant()` works downstream | `Account` | Implicit privilege — worth interrogating |
| 6 | View | `service_api/app/completion.py` | Same `AppGenerateService.generate` path as Flow 1, `invoke_from=SERVICE_API` | — | Published (not draft) workflow selected |

Validation and authorization: the whole decorator *is* the boundary. Note what it does **not** check: the `user` field is chosen by the API caller — end users are identifiers scoped to the app, not authenticated principals. That's a documented design choice (the token holder owns its user namespace), but juniors often mistake it for IDOR.

What seniors notice: kwargs injection (`kwargs["app_model"] = app_model`, `#L79`) couples decorator and view signatures; the owner-account fallback (step 5) means service-API actions can execute with owner-level identity — blast radius to audit when adding new service-API endpoints.

Interview angle: "How would you design API keys for a multi-tenant product?" Use: token → app → tenant chain of checks, caching with single-flight, and the end-user namespace tradeoff.

Drill: diff this decorator against the console stack from Flow 1 — write a two-column table of every check each performs. Strong = you can say which endpoint type would catch a *revoked* token faster and why.

---

## Flow 6: Frontend data fetching & cache invalidation (client flow)

Why this flow matters: this is your daily-driver stack (TanStack Query) done at production scale.

Open these files first:
- `web/service/use-apps.ts#L30-L81`
- `web/service/fetch.ts#L122-L249`
- `web/service/client.ts` — typed oRPC-style console client (`consoleClient`, `consoleQuery`)

Trace:

| Step | Owner | File | What happens | Data shape | Risk |
| --- | --- | --- | --- | --- | --- |
| 1 | Hook | `use-apps.ts#L43-L49` | `useQuery({queryKey: ['apps','detail',id], queryFn: get(...), enabled: !!id})` | `App` | Key discipline = cache correctness |
| 2 | HTTP | `fetch.ts#L137-L249` | ky client: base URL by API class (console/public/marketplace), CSRF header, JSON parsing, blob detection | typed `T` | 204→success normalization hook (`#L26-L33`) |
| 3 | Errors | `fetch.ts#L61-L79` | Non-2xx: parse error body, toast unless silent/401, special-case `already_setup` redirect | `ResponseError` | Global side effects (toast) in the HTTP layer |
| 4 | Mutation | `use-apps.ts#L60-L81` | `useMutation` + `onSuccess`: invalidate both the typed query key *and* a legacy local key | — | Two key systems coexist (migration in progress) — miss one and lists go stale |

What juniors usually miss: `enabled: !!appID` prevents firing with undefined params; invalidation is *targeted* (per-key), not "refetch everything."

What seniors notice: the codebase is mid-migration from hand-rolled service functions (`service/apps.ts`) to a generated/typed client (`service/client.ts`, `consoleQuery.apps.list.key()`); every new feature must pick the new pattern — a live example of incremental migration strategy.

Interview angle: "How do you keep client cache consistent after writes?" — answer: query-key taxonomy + invalidation on mutation, show the double-invalidate here, and name the failure mode (missed invalidation → stale UI) plus the mitigation (centralized key factories).

Drill: find one more mutation in `web/service/` that invalidates queries and one that (apparently) should but doesn't. Write the bug report you'd file for the latter. Strong = your report names the exact stale screen a user would see.
