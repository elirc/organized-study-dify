# Pattern Catalog

Fourteen cards. The goal is **recognition** — seeing the shape in unfamiliar code — not name-dropping. Each card: problem → shape → real anchors → failure modes → when (not) to use → interview angle → drill.

---

## Pattern 1: Application Factory

Problem it solves: constructing the app with config/extensions varying by context (web, worker, migrations, tests) without global import-time side effects.
General shape: `create_app(config) -> app`; extensions get `init_app(app)`.
Real example: `api/app_factory.py#L49-L138`; minimal variant for migrations `#L222-L230`.
Second example: `celery_entrypoint.py` boots the same factory for workers.
Why this works: explicit ordered extension list (`#L177-L206`) with per-extension `is_enabled()` gates and timing logs.
Failure modes: import-time cycles force lazy imports inside functions (visible throughout); forgetting an extension in a secondary entrypoint (worker without `ext_storage` → runtime crash).
Use when: any app with multiple entrypoints. Avoid when: single-entry scripts — ceremony.
Interview angle: "How do you structure a Flask/Express app for testability?"
Drill: list which extensions `create_migrations_app` skips and why each would be wrong to load.

## Pattern 2: Decorator-Chain Authorization (middleware pipeline)

Problem: per-route composition of authn/authz/resource-loading without repeating code.
Shape: ordered decorators, each raising or enriching kwargs.
Real: `completion.py#L155-L159`; `service_api/wraps.py#L55-L120`.
Second: `console/wraps.py` billing/rate-limit decorators (L94, L161).
Failure modes: wrong order (authz before authn), kwargs-injection breaking on rename, dead decorators nobody removes.
Use when: cross-cutting checks per route. Avoid when: logic needs data from the body — that belongs in the service.
Interview angle: "Implement role-based access control cleanly."
Drill: write the ASCII pipeline for `ChatMessageApi.post` with what each stage can reject.

## Pattern 3: Strategy Dispatch by Enum (App Modes)

Problem: one endpoint, N behaviors (chat/completion/agent/workflow…).
Shape: `match mode: case X: XGenerator().generate(...)` — each strategy a class with a common interface.
Real: `app_generate_service.py#L124-L263`.
Second: `IndexProcessorFactory(index_type).init_index_processor()` (`indexing_runner.py#L93`).
Failure modes: default-case blindness (`case _: raise`, good here); strategies drifting apart in capabilities (see how ADVANCED_CHAT/WORKFLOW branches carry extra streaming machinery the others don't).
Use when: variants share a contract. Avoid when: two variants — an `if` is fine.
Interview angle: "Refactor this if/else ladder."
Drill: list what all six branches have in common and extract (on paper) the shared pre/post steps.

## Pattern 4: Reserve → Commit → Refund (compensating quota)

Problem: enforce quotas for work that may fail after the check.
Shape: reserve capacity; do work; commit on success / refund on error.
Real: `app_generate_service.py#L107-L120` (reserve+commit), `#L264-L267` (refund in except).
Failure modes: refund path missed on some exceptions; process death = leaked reservation (needs TTL).
Interview angle: "How do you rate-limit/bill accurately for failable operations?"
Drill: find where `RateLimit.exit` runs for streaming vs blocking (`#L268-L270`) and explain the asymmetry.

## Pattern 5: Subscribe-Gated Task Start (rendezvous)

Problem: at-most-once transport loses events emitted before the consumer subscribes.
Shape: enqueue producer on first subscribe; fallback timer so an absent consumer can't block work forever; lock guarantees exactly-one start.
Real: `app_generate_service.py#L41-L85`.
Failure modes: fallback window drops early events on pub/sub (accepted); lock scope bugs.
Interview angle: "Design progress streaming for a job queue."
Drill: rewrite it (paper) assuming Redis Streams only — how much code disappears?

## Pattern 6: Durable Event Log + Replayable Subscription

Problem: browser disconnects mid-job; user reloads; second tab.
Shape: producer appends events keyed by run-id; consumers replay from offset.
Real: run id minted pre-enqueue `app_generate_service.py#L159-L172`; frontend re-subscribe on pause `chat/hooks.ts#L1140-L1147`.
Failure modes: unbounded streams (need TTL/trim), replay vs live-tail race.
Interview angle: classic system-design: "notifications/progress with reconnect."

## Pattern 7: Tenant-Isolated Work Queue (fairness scheduler)

Problem: shared Celery queue; one tenant's 10k-doc upload starves everyone.
Shape: per-tenant Redis list + running flag; completion pulls next batch for that tenant.
Real: `document_indexing_task.py#L192-L231`.
Failure modes: flag/list protocol races on worker death; per-tenant concurrency config drift.
Interview angle: "Multi-tenant fairness in background processing" — a differentiating answer.

## Pattern 8: Two-Phase Status Transaction (short tx, long work)

Problem: long pipelines must not hold DB transactions.
Shape: tx1: set status=IN_PROGRESS, commit; work sessionless; tx2: set terminal status.
Real: `document_indexing_task.py#L93-L120`; error terminalization `indexing_runner.py#L58-L67`.
Failure modes: crash mid-work leaves IN_PROGRESS forever → needs sweeper/timeout.
Interview angle: "How do you track long job state?"
Drill: find whether anything sweeps stale `PARSING` documents (`api/schedule/`); write the sweeper spec if not.

## Pattern 9: Versioning by Pointer (draft/published)

Problem: edit safely while a stable version serves traffic.
Shape: immutable version rows + a mutable pointer (+ a special draft row).
Real: `App.workflow_id` pointer `model.py#L405`; draft-vs-published fetch by `InvokeFrom` (`app_generate_service.py#L386-L420`).
Failure modes: pointer swung before version validated; deleting versions still pointed at.
Interview angle: "Design content publishing with rollback" — pointer swap = O(1) rollback.

## Pattern 10: Single-Flight Cached Lookup

Problem: hot lookup (API token) hammering the DB; cache stampede on expiry.
Shape: cache + "only one caller populates on miss, others wait."
Real: `fetch_token_with_single_flight` used at `service_api/wraps.py#L23`, `L61` (impl in `services/api_token_service.py`).
Failure modes: stale-after-revoke (security-relevant here!), lock starvation.
Interview angle: "Cache stampede — what is it, how do you stop it?"

## Pattern 11: LocalProxy / Ambient Context

Problem: passing current-user through every signature is noisy.
Shape: context-local proxy resolved per request (`current_user`), with explicit accessors for the typed version.
Real: `libs/login.py#L105-L118`; `current_account_with_tenant()` `#L38-L48`; worker-side analogue `RecyclableContextVar` (`app_factory.py#L59-L63`).
Failure modes: ambient state in code that later runs in workers (no request context); type ambiguity (Account vs EndUser) — hence the isinstance asserts.
Interview angle: Node's `AsyncLocalStorage` is the same idea — cross-stack fluency point.

## Pattern 12: Layered Engine Configuration (middleware for engines)

Problem: bolt cross-cutting concerns (limits, quota, logging, tracing) onto an execution engine without forking it.
Shape: `engine.layer(X).layer(Y)` — observer/decorator hybrid.
Real: `workflow_entry.py#L199-L220` (DebugLogging, ExecutionLimits, LLMQuota, Observability layers).
Failure modes: layer ordering sensitivity; layers doing heavy work per node event.
Interview angle: "How would you add per-customer limits to an engine you don't own?"

## Pattern 13: Typed HTTP Client Factory + Hook Composition (frontend)

Problem: every request needs base URL, credentials, CSRF, error policy — without repeating.
Shape: one configured client (`ky.create`), per-call `extend` with hooks; thin typed wrappers per resource.
Real: `web/service/fetch.ts#L122-L205`; consumers `web/service/apps.ts`.
Failure modes: policy hidden in the client surprises callers (global toasts); bypassing the client (raw fetch) skips auth/CSRF.
Interview angle: "Structure the API layer of a large React app."

## Pattern 14: Optimistic UI with Placeholder Reconciliation

Problem: chat must feel instant; server assigns real ids later.
Shape: insert placeholder nodes with temp ids; stream mutates; reconcile ids on completion.
Real: `chat/hooks.ts#L644-L679` (placeholders), stream callbacks `#L1110-L1155`.
Failure modes: error paths leaving orphan placeholders; temp-id collisions (`Date.now()`); reconciliation with tree branching.
Interview angle: "Implement optimistic updates — what happens on failure?"
Drill: find where a failed send removes (or fails to remove) the placeholder pair. Write the repro if it doesn't.

---

**Recognition drill (whole catalog)**: pick any OSS repo you know (e.g., Excalidraw, cal.com). For five cards above, find the analogous code there or state confidently that the pattern is absent and why. Strong = you can name the *forces* (scale, tenancy, transport) that make Dify need patterns your other repo doesn't.
