# Trace Tables

Fill each table yourself first (columns: step, file/line, value shape, owner, transformation, risk). Reference rows provided; your job is to verify and extend them by opening the files.

---

## Trace 1 — UI to API: the `query` string of a chat message

| Step | File:line | Value shape | Owner | Transformation | Risk |
| --- | --- | --- | --- | --- | --- |
| 1 | chat input component → `handleSend` arg | `string` | React | none | empty-string handling? |
| 2 | `chat/hooks.ts#L685-L692` | field of `bodyParams` | hook | merged with conversation id, files, inputs | inputs processed via `getProcessedInputs` |
| 3 | `base.ts#L517-L519` | JSON string | HTTP layer | `JSON.stringify` | encoding |
| 4 | `completion.py#L161-L162` | `ChatMessagePayload.query` | Pydantic | validated (`Field(...)` required) | max length? (check) |
| 5 | `app_generate_service.py#L89-L97` | `args["query"]` in Mapping | service | passed through | — |
| 6 | core app runner → prompt assembly | prompt message part | core | templating, history joining | prompt injection is *by design* here — the platform's whole point; moderation layer exists (`core/moderation/`) |
| 7 | `Message` row (`model.py#L1393`) | `query` column | DB | persisted | PII retention |

Extension task: add a row for where `query` is embedded for retrieval when the app has datasets attached (`core/rag/retrieval/dataset_retrieval.py`).

## Trace 2 — Persistence: a document's `indexing_status`

| Step | File:line | Value | Owner | Risk |
| --- | --- | --- | --- | --- |
| 1 | creation (`DocumentService.build_document`, dataset_service) | `waiting` (check default) | service | — |
| 2 | `document_indexing_task.py#L101-L105` | `parsing` + `processing_started_at` | task (short tx) | crash later → stuck `parsing` |
| 3 | `indexing_runner.py` `_extract/_transform/_load_segments` | `splitting`/`indexing` transitions (verify inside those methods) | runner | partial progress |
| 4 | `_load` completion | `completed` | runner | — |
| 5 | any exception | `error` + `error` text + `stopped_at` (`indexing_runner.py#L58-L67`) | runner | error strings user-visible — leakage of internals? |

Your task: open the runner's private methods and complete the exact transition list with line anchors; then draw the legal state machine and mark unreachable/terminal states.

## Trace 3 — Auth: from `Authorization: Bearer app-…` to `current_user` in a service-API view

| Step | File:line | Value | Risk |
| --- | --- | --- | --- |
| 1 | `validate_and_get_api_token("app")` (`service_api/wraps.py#L61`) | `ApiToken` | cache staleness on revoke |
| 2 | `#L63-L71` | `App` (status/enable_api checked) | — |
| 3 | `#L73-L77` | `Tenant` (not archived) | — |
| 4a | with `fetch_user_arg`: `#L82-L103` | `EndUser` bound to flask-login | caller-chosen identity namespace |
| 4b | without: `#L104-L120` | tenant **owner** `Account` logged in | privilege breadth |
| 5 | view body | `current_user` LocalProxy resolves | Account-vs-EndUser type confusion |

Your task: add the console-API twin rows (`libs/login.py`) and mark the *first* line where the two paths diverge.

## Trace 4 — Error: provider failure during chat, server → toast

| Step | File:line | Shape |
| --- | --- | --- |
| 1 | provider call raises `InvokeError` (graphon runtime) | exception |
| 2 | `completion.py#L194-L195` | `CompletionRequestError(e.description)` — HTTP-coded error |
| 3 | flask-restx error handler | JSON `{code, message, status}` |
| 4 | streaming case: error event in SSE (`base.ts` handleStream `moreInfo.errorMessage`) | SSE event |
| 5 | `base.ts#L557-L564` | `onError(msg)` + toast (with string-match suppression of abort noise) |
| 6 | `chat/hooks.ts` onError handler | placeholder answer marked errored (verify how) |

Your task: fill step 6 precisely, then answer: which step *loses* the error `code`? (Grep whether `errorCode` survives to the UI decision.)

## Trace 5 — Cache: app list after "delete app"

| Step | File:line | What changes |
| --- | --- | --- |
| 1 | `use-apps.ts#L63-L69` | DELETE via typed client |
| 2 | server: `services/app_service.py` delete path | rows + related resources |
| 3 | `use-apps.ts#L70-L78` | invalidate `consoleQuery.apps.list.key()` **and** legacy `['apps','full-list']` |
| 4 | TanStack Query | active queries refetch |

Your task: find one other app mutation (rename? copy?) and verify both key systems are invalidated. Record a gap if not.

---

**Rubric for all traces** — Basic: every row has a real file:line. Solid: transformations correct; you found at least one row the reference table got vague and made it precise. Strong: each risk cell contains a *testable* statement, and you can present any trace aloud in <2 minutes (that's the debugging-interview format — see [08/05](../08-interview-prep/05-debugging-and-code-review-rounds.md)).
