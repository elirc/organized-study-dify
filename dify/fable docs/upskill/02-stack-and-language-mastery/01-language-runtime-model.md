# Language & Runtime Models

## The browser side: one thread, an event loop, and promises

Mental model: JS runs one call stack; async work (fetch, timers) completes via task/microtask queues. Promises resolve in microtasks; `await` is syntactic suspension, not parallelism. Parallelism requires *concurrent I/O* (`Promise.all`) or workers.

Where this repo uses it well:

- **Streaming without blocking**: `ssePost` (`web/service/base.ts#L452-L580`) never `await`s the whole response — it hands the `Response` to `handleStream`, which reads the body incrementally and fires callbacks per SSE event. The UI thread stays free; React re-renders per chunk.
- **Fire-and-forget with delivery guarantee**: `postWithKeepalive` (`web/service/fetch.ts#L257-L275`) uses `keepalive: true` so a POST survives page unload — the only correct way to send "user left" beacons; a plain `fetch` would be cancelled.
- **Cancellation**: `AbortController` is created per stream and handed out via `getAbortController` (`base.ts#L493-L513`); the chat hook aborts the *previous* workflow-events subscription before opening a new one (`chat/hooks.ts#L1158-L1160`). Cancellation-as-a-value is a senior habit: whoever starts async work must own stopping it.

Sharp edges to watch here:

- The 401 handler retries `ssePost` recursively after a token refresh (`base.ts#L540-L545`). There is no visible retry cap — reason about what happens if refresh keeps "succeeding" while the API keeps rejecting. (Labeled *investigate*, not bug.)
- Error filtering by string matching (`base.ts#L561`) is fragile across browsers — a good "what would you improve" interview example.

Pitfall checklist (JS): forgetting that `await` in a loop serializes I/O; assuming `finally` runs on tab close; mutating React state from a stream callback without functional updates; leaking AbortControllers.

## The server side: processes and threads, not an event loop

Flask under gunicorn (`api/gunicorn.conf.py`) is the opposite model: each request occupies a worker (process/thread/greenlet per config). There's no single event loop to starve — but blocking is still costly because workers are finite.

Where this repo confronts it:

- **Long work leaves the request**: workflow execution moves to Celery (`app_generate_service.py#L156-L185`) precisely because holding a gunicorn worker for a multi-minute graph run would exhaust the pool. The request thread only *subscribes to events*.
- **Threads used sparingly and defensively**: the subscribe-gated task start uses a `threading.Lock` + `threading.Timer` (`app_generate_service.py#L49-L85`) to guarantee exactly-one enqueue between "first subscriber" and "200ms fallback." Read it as a model of writing race-safe code: a `started` flag only ever checked/flipped under the lock.
- **Context is explicit, not ambient**: worker code can't touch `flask.request`. The workflow engine captures context up front (`workflow_entry.py#L182-L183`) and `contexts/` provides `RecyclableContextVar` re-initialized per request (`app_factory.py#L59-L63`) — Python's `contextvars` playing the role that AsyncLocalStorage plays in Node.
- **The GIL doesn't matter here as much as you think**: the heavy work is I/O (LLM APIs, DB, vector stores). The graph engine runs nodes with a worker pool whose size is config-bounded (`workflow_entry.py#L190-L195`).

Pitfall checklist (Python/Flask): using module-level mutable state across requests; assuming a DB session survives into a Celery task (it doesn't — tasks re-query by id, see `document_indexing_task.py#L59-L63`); forgetting `db.session` is request-scoped while `Session(db.engine)` context managers are explicit (`api/AGENTS.md` SQLAlchemy section).

## Serial vs parallel work — one rule, two stacks

The question is always: *what is the unit that blocks, and what is the budget?* Browser: the main thread, budget ~16ms/frame → stream + callbacks. Flask: worker slots, budget ~seconds → offload to Celery. Celery: queue throughput per tenant, budget = fairness → tenant-isolated queues (`document_indexing_task.py#L192-L231`).

## Drills

1. In `chat/hooks.ts#L620-L709`, list every piece of state that changes before the first byte arrives. What re-renders? (Rubric — Strong: you mention the tree update helper and why placeholders use `Date.now()` ids.)
2. Explain why `_build_streaming_task_on_subscribe` needs both the lock *and* the timer. Write the interleaving that breaks with only one. (Strong: you produce a timeline with two threads.)

## Interview angle

1. "Explain the event loop / microtasks vs macrotasks." → Anchor: SSE streaming keeps the loop free; then contrast with `await`-in-loop serialization. See [08-interview-prep/01-js-ts-node-deep-dive.md](../08-interview-prep/01-js-ts-node-deep-dive.md) Q1–Q3.
2. "How do you cancel in-flight requests when the user navigates?" → `AbortController` handling in `base.ts`/`chat/hooks.ts`. (08/Q4)
3. "Your endpoint takes 90s. What do you do?" → Celery offload + event replay (Flow 2). (08/04-system-design)
4. "What's a race condition you've actually reasoned about?" → the subscribe-gate lock/timer. (08/Q5)
