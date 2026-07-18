# Systematic Debugging

The loop: **reproduce → narrow (bisect layers) → hypothesize → test cheaply → fix root cause → add regression coverage**. The discipline is refusing to "try things" before you can predict what a probe will show.

Tools in this repo: browser devtools (Network → EventStream tab for SSE), React DevTools, TanStack Query devtools (`@tanstack/react-query-devtools` in catalog), Flask debug logs (`DEBUG` config prints extension timing, `app_factory.py#L136-L137`), `DebugLoggingLayer` for workflow runs (`workflow_entry.py#L199-L209`), SQLAlchemy echo, Celery worker logs, Redis CLI (`MONITOR`, `LRANGE` on tenant queues), and status columns in the DB itself.

---

## Scenario 1: "My document is stuck on *parsing* forever"

Reproduction: upload a doc; kill the Celery worker mid-run; status never changes.
First question: is the bug in **dispatch** (task never ran) or **execution** (task died mid-way)?
Narrowing path:
1. Check DB: `processing_started_at` set? (`document_indexing_task.py#L104`) — set ⇒ task started ⇒ execution problem.
2. Not set ⇒ check Redis/Celery: was the task enqueued? Any worker consuming queue `dataset`? Check the tenant-isolation flag — a stale "running" flag parks new work in the tenant list (`#L206-L231`).
3. Started ⇒ worker logs around the crash; check for exceptions swallowed into `Document.error` (should be terminal ERROR — if not, the crash was hard).
Useful probes: `redis-cli LRANGE` the tenant queue key; `SELECT indexing_status, error, processing_started_at FROM documents WHERE id=…`.
Likely root causes: dead worker with stale tenant-queue flag; billing-check exception path; storage fetch hang.
Regression test to add: sweeper test (stale PARSING > N hours ⇒ ERROR) — which first requires *building* the sweeper (see critique #2).
Senior lesson: long-running pipelines need a liveness invariant, not just status writes.
Interview version: narrate as "I bisected dispatch vs execution using the timestamp column the code writes at phase 1 — data-model breadcrumbs beat log spelunking."

## Scenario 2: "Chat streams the answer but the UI shows nothing until the end"

Reproduction: messages arrive in one burst at stream end.
First question: server not flushing, proxy buffering, or client not processing incrementally?
Narrowing path:
1. `curl -N` the endpoint (or devtools EventStream): chunks arriving over time? Yes ⇒ client-side. No ⇒ server/proxy.
2. Server side: is the response actually SSE (`compact_generate_response`, used at `completion.py#L99/L178`)? Streaming flag true (`response_mode` handling `#L164`)?
3. Proxy: dev proxy / nginx buffering SSE? (Classic: missing `X-Accel-Buffering: no` or compression middleware buffering — note `ext_compress` is in the extension list, `app_factory.py#L184` — does it exempt event streams?)
4. Client: breakpoints in `handleStream` reader loop (`base.ts#L189+`).
Likely root causes: compression/proxy buffering; blocking mode accidentally selected.
Regression: E2E asserting first token latency << total time.
Senior lesson: streaming bugs are usually *between* components; test each hop with the narrowest tool (curl beats console.log).

## Scenario 3: "Deleted an app; it still shows in the list until hard refresh"

First question: server didn't delete, or client cache stale?
Narrowing: 1) Network tab — DELETE 200? 2) Refetch after mutation fired? Query devtools: which keys invalidated. 3) Compare against `use-apps.ts#L70-L78` — during the key-system migration, a screen may read the *legacy* key while the mutation invalidates only the new one (or vice versa).
Likely root cause: missed invalidation of one of the two key systems.
Regression: hook test asserting both keys invalidated (Recipe 8).
Senior lesson: during migrations, bugs cluster at the seam; grep for *all* readers of the stale data before fixing the one you found.

## Scenario 4: "Service API returns 401 for a token that works in another environment"

First question: which check rejected it? The chain has distinct messages: token lookup, app status, `enable_api`, tenant archive (`service_api/wraps.py#L61-L77`).
Narrowing: read the error body/code; if generic, add temporary structured logging per stage (tenant/app ids — house rule, `api/AGENTS.md`). Check the token cache (single-flight) — a *revoked-then-recreated* token can be stale-cached.
Likely root causes: wrong environment's SECRET/DB; app API disabled; cache staleness.
Regression: unit tests per rejection branch with distinct error codes (if codes are indistinct, that's the *real* fix — make the boundary observable).
Senior lesson: a boundary that can reject for five reasons must say which one — error specificity is an operability feature.

## Scenario 5: "Workflow run hangs at a node; UI spinner forever"

First question: is the engine stuck, or did it finish and delivery fail?
Narrowing: 1) DB: `WorkflowRun` status? Finished ⇒ delivery problem: check event stream in Redis (streams: entries exist? pub/sub: was anyone subscribed when they fired?). 2) Running ⇒ engine: enable `DebugLoggingLayer` (DEBUG mode adds it per run, `workflow_entry.py#L200-L209`); check `ExecutionLimitsLayer` bounds — max steps/time exceeded should end the run (`#L212-L215`). 3) Check for the pause state — human-input nodes legitimately pause (frontend then re-subscribes, `chat/hooks.ts#L1140-L1147`).
Likely root causes: pub/sub transport + subscriber died (events lost); node awaiting an external tool with no timeout.
Regression: integration test with a deliberately slow node asserting limit enforcement.
Senior lesson: separate "compute state" from "delivery state" in your head *before* probing — they fail independently by design here.

---

**Drill**: run scenario 3 for real: comment out one invalidation locally, reproduce, then restore. Feeling the bug beats reading about it.

**Interview note**: in debugging rounds, narrate the *decision tree*, not the outcome: "two hypotheses; the cheapest discriminating probe is X; result eliminates hypothesis A" — see [08/05](../08-interview-prep/05-debugging-and-code-review-rounds.md) for the timed versions of scenarios 1, 3, and 5.
