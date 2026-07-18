# Type Systems and Contracts

A **contract** is any shape two parties rely on without reading each other's code: an HTTP payload, a DB row, a task's arguments, a component's props. Types are how you *write contracts down*; validation is how you *enforce them at runtime*. This repo shows both, twice (TS and Python).

## TypeScript side

- **Strict config + no `any`** is house law (`AGENTS.md`, `web/AGENTS.md`). The service layer returns typed generics: `base<T>(url, …): Promise<T>` (`web/service/fetch.ts#L137`), so callers declare the response shape (`get<App>('/apps/…')`, `use-apps.ts#L46`).
- **Discriminated callbacks as a contract**: the SSE layer types ~30 event payloads (`web/service/base.ts#L57-L132`) — `NodeStartedResponse`, `WorkflowFinishedResponse`, etc. from `@/types/workflow`. That's a union-shaped protocol between backend event emitter and frontend UI. When the backend adds an event type, TS won't catch it (the wire is untyped JSON) — which is why the repo is moving toward a **generated typed client** (`web/service/client.ts`, `@hey-api/openapi-ts` in the catalog, backend OpenAPI under `api/openapi/`). Contract drift between hand-written types and the wire is the #1 fullstack bug class; codegen is the mid-level answer, contract tests the senior addition.
- **`unknown` over `any` at boundaries**: error parsing does `const data: unknown = await response.clone().json()` then narrows (`fetch.ts#L66-L67`) — the correct pattern: parse to `unknown`, validate/narrow, then use.

## Python side: Pydantic v2 as the runtime type system

- **Request contracts**: controllers validate with Pydantic models — `ChatMessagePayload` with field validators for UUIDs (`api/controllers/console/app/completion.py#L55-L68`); the alias `model_config` → `model_config_data` (`#L45`) exists because `model_config` is reserved by Pydantic itself — a nice "leaky abstraction" anecdote.
- **Task contracts**: Celery payloads are Pydantic too — `AppExecutionParams` is serialized with `model_dump_json()` before `.delay(payload_json)` (`app_generate_service.py#L159-L172`). Versioning discipline: a worker running old code must still parse new payloads (or you drain queues before deploy) — say this in interviews; it's the async twin of API versioning.
- **TypedDict for known-shape dicts**: house style prefers `TypedDict` over loose `dict` (`api/AGENTS.md`), visible en masse in `api/models/model.py#L90-L343` (config dicts) and `workflow_entry.py#L119-L136` (graph config shapes).
- **Typed ORM rows**: SQLAlchemy 2.0 `Mapped[...]`/`mapped_column` gives static types on models (`App`, `model.py#L392-L424`; `Account`, `account.py#L87-L119`). Enums are stored as text via `EnumText(AppMode)` — the DB stores strings, Python sees enums; the *migration* is the contract change.

## Where contracts live end-to-end (one field's journey)

Trace `conversation_id`: TS type in chat body (`chat/hooks.ts#L685-L692`) → JSON → Pydantic `ChatMessagePayload.conversation_id` with UUID validator (`completion.py#L57-L65`) → service arg mapping → `Conversation` row (`models/model.py#L1044`). Four representations, three conversion points, each a place to validate or to drift. That sentence — "every field has N representations and N−1 conversion points" — is a strong interview line.

## Failure modes

- Hand-written TS types drift from API reality → runtime `undefined` deep in a component. Mitigation here: codegen migration; your mitigation anywhere: zod-parse at the boundary.
- Pydantic `model_dump(exclude_none=True, by_alias=True)` (`completion.py#L162`) changes key names/presence — downstream `dict.get` chains silently miss renamed keys.
- Enum stored as free text: a bad row (manual SQL, old migration) crashes on enum parse — defensive `AppMode.value_of` exists for this.

## Drills

1. Add (on paper) a `priority: 'low'|'normal'` field to chat messages. List every file from the trace above you'd touch, in order, and which layer rejects a bad value first.
2. Find one place `as` casting in `web/service/` hides a real risk, and rewrite it with narrowing. Rubric — Strong: your rewrite fails loudly on drift instead of silently.
3. In `api/models/dataset.py`, compare `Dataset.retrieval_model` (JSON blob) vs proper columns. When is a JSON column the right call? (Answer shape: write-mostly config with evolving schema vs queryable/indexable data.)

## Interview angle

1. "`unknown` vs `any`?" → `fetch.ts#L66` narrowing example. (→ 08/01)
2. "How do you keep FE/BE types in sync?" → drift story + codegen migration in this repo. (→ 08/03)
3. "Runtime validation vs static types?" → Pydantic at every boundary (HTTP, queue); static types can't defend a wire. (→ 08/01, 08/03)
4. "Schema design: JSON column vs table?" → `retrieval_model` example. (→ 08/03)
