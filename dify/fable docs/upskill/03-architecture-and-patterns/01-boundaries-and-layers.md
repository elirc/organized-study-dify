# Boundaries and Layers

## The declared architecture

`api/AGENTS.md` states it: *controller → service → core/domain*, and "Controllers: parse input via Pydantic, invoke services, return serialised responses; no business logic." Layers, top to bottom:

| Layer | Owns | Must not own |
| --- | --- | --- |
| Controllers (`api/controllers/`) | HTTP shape, auth decorators, error→status translation | business rules, direct model mutation |
| Services (`api/services/`) | use-cases, permission checks, task dispatch, transactions | HTTP objects, response formatting |
| Core (`api/core/`) | domain engines (app generation, workflow, RAG, tools) | Flask request context, controller imports |
| Models (`api/models/`) | table shape, trivial row helpers | orchestration, network calls |
| Tasks (`api/tasks/`) | Celery entrypoints — thin wrappers that re-validate and call services/core | novel business logic |

Enforcement isn't just convention: **import-linter** runs in `make lint` (Makefile L77-L83), turning layer rules into CI failures. That's the senior move — architecture as an executable check.

## Good boundaries, with evidence

- **Error translation at the edge**: domain exceptions (`ProviderTokenNotInitError`, `QuotaExceededError`) are raised deep and converted to HTTP errors only in the controller (`api/controllers/console/app/completion.py#L179-L200`). Core code never knows HTTP exists.
- **Queue boundary passes ids, not objects**: `document_indexing_task.py#L33` receives `dataset_id, document_ids`; the task re-queries and re-validates (`#L59-L91`). The DB row is the contract, not a pickled object.
- **Explicit context capture**: core workflow code that must run in both web and worker contexts captures request context explicitly (`workflow_entry.py#L182-L183`) instead of reaching for Flask globals.
- **Frontend service layer**: components never call `fetch`; they call `web/service/*` functions or query hooks, so cross-cutting concerns (CSRF `fetch.ts#L184`, 401 refresh `base.ts#L540-L545`, toasts) live in one place.

## Boundary leaks, with evidence (learn to *spot*, then to *articulate*)

1. **Model layer doing business writes**: `App.is_agent` — a property getter — mutates `self.mode` and calls `db.session.commit()` (`api/models/model.py#L471-L484`). A read (`if app.is_agent`) can silently write the DB mid-request, inside whatever transaction happens to be open. Why it exists: lazy migration of legacy chat apps into agent-chat. Why it's a leak: ownership violation (models own shape, services own transitions) + surprise side effect + hidden commit. How you'd fix it: an explicit migration or a service-level normalization on read paths, and make the property pure.
2. **Models issuing their own sessions**: `Account.current_tenant` setter opens a fresh `Session(db.engine)` (`account.py#L129-L150`), and `App.site`/`app_model_config` properties run queries via `db.session` (`model.py#L441-L450`). Convenient, but N+1-prone (every property access = query) and it hides I/O inside attribute access — the reader can't see the cost.
3. **Decorator→view kwargs injection**: `get_app_model` and `validate_app_token` inject `app_model`/`end_user` into view kwargs (`service_api/wraps.py#L79-L99`). Pragmatic Flask idiom, but the contract is invisible in the view signature; renaming a kwarg breaks at runtime, not type-check time.
4. **HTTP-layer UI side effects**: the frontend fetch layer toasts errors itself (`fetch.ts#L61-L79`). Convenient globally, but the *service layer decides UX* — screens that need custom error handling must remember `silent: true`. Boundary question to ask anywhere: "who owns the decision to show the user something?"

None of these are emergencies; each is a tradeoff someone chose under pressure. The skill is naming the tradeoff without moralizing — exactly what a code-review or interview answer needs.

## How to check a boundary in any repo (portable checklist)

1. Pick a layer pair (controller/service). Grep imports upward and downward — who imports whom?
2. Find state mutations: which layer calls `commit()`? (Here: services and tasks mostly; the `is_agent` property is the outlier.)
3. Find the error vocabulary: does the inner layer raise domain errors or framework errors?
4. Ask "could this inner function run in a test/worker with no HTTP?" If not, the boundary is leaky.

## Drill

Take `api/services/app_generate_service.py#L87-L270` and mark every line that touches: HTTP concepts (should be none), DB, Redis, Celery, config. Rubric — Basic: correct markings. Solid: you notice the service knows about SSE transport config (`PUBSUB_REDIS_CHANNEL_TYPE`, `#L65`) and can argue whether that's a leak. Strong: you propose where transport choice *should* live and what interface would hide it, plus the migration cost.

## Interview angle

"Describe a layering violation you've found and how you'd fix it" → the `is_agent` property story: what it does, why it's risky (hidden write, ownership), the fix, and the migration plan (backfill + pure property). That's a complete mid-to-senior answer with evidence. (→ [08-interview-prep/03](../08-interview-prep/03-api-and-data-modeling-questions.md), [08/06 STAR stories](../08-interview-prep/06-behavioral-star-stories.md))
