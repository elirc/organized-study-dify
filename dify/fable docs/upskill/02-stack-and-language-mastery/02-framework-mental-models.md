# Framework Mental Models

## React 19: render is a function of state; commit is when the world changes

Model: components re-render when their state/props/context change; the render must be pure; effects run after commit. Everything else (memoization, keys, stale closures) follows.

In this repo:

- **State placement**: the chat keeps its message *tree* in the `useChat` hook's local state (`web/app/components/base/chat/chat/hooks.ts#L55+`), while the workflow editor keeps graph state in **Zustand slices** (`web/app/components/workflow/store/workflow/` — `node-slice.ts`, `panel-slice.ts`, `history-slice.ts`, …). The rule the repo demonstrates: local, ephemeral, single-owner state → hook state; cross-component, tool-like state (a whole canvas) → external store. Server data → TanStack Query, never copied into either.
- **Stale closures in the wild**: `handleSend`'s `useCallback` dependency array (`chat/hooks.ts#L1170-L1181`) is ten entries long. Every missed dependency is a stale-closure bug; every unnecessary one is a re-created callback. When a streaming callback needs *latest* state without re-subscribing, the code uses refs (`isRespondingRef`, `pausedStateRef` — grep them). Ref-for-latest-value is the standard escape hatch; name it in interviews.
- **Mutation-then-set anti-pattern to watch**: stream handlers mutate `responseItem` in place then call tree-update helpers (`#L1110-L1155`). It works because the helper clones into state, but it's a good review-kata topic: where exactly does mutable data become immutable state?

## TanStack Query: server state is a cache, not your state

Model: every server read is `(queryKey → queryFn)` in a normalized cache with staleness; writes are mutations that *invalidate* keys. You never `useEffect`+`fetch`+`setState` again.

In this repo: `web/service/use-apps.ts#L43-L49` (keys, `enabled`), invalidation on delete (`#L60-L81`), and a live migration toward generated typed clients + key factories (`service/client.ts`, `consoleQuery.apps.list.key()`). Failure mode: forget an invalidation → stale list; the delete mutation invalidates **two** keys because two key systems coexist during migration.

## Zustand: a store is just a closure with subscriptions

Model: `create()` returns a hook; slices compose into one store; components subscribe to *selected* slivers so unrelated updates don't re-render them. Compare to Redux: no actions/reducers ceremony; discipline must come from slice boundaries instead.

In this repo: the workflow editor composes ~15 slices (`store/workflow/index.ts`) — node state, panel state, undo history (`zundo` is in the catalog), env vars, versions. Drill: open `node-slice.ts` and find one selector-based subscription in `web/app/components/workflow/nodes/`; explain what re-renders when a node moves.

## Flask: decorators are middleware; blueprints are routers

Map from Express: `@console_ns.route` ≈ `router.post(...)`; the decorator stack ≈ per-route middleware chain (`completion.py#L146-L160`); `before_request` ≈ app-level middleware (`app_factory.py#L59-L96`); blueprints/namespaces ≈ routers mounted under prefixes (`extensions/ext_blueprints.py`); error translation in views ≈ centralized error middleware, but done per-controller here with domain-exception catches (`completion.py#L179-L200`).

Where it's sharp: decorator *order* is the whole security model — `login_required` must precede role checks; `get_app_model` injects `app_model` into kwargs, which means view signatures silently depend on decorators (also done at `service_api/wraps.py#L79`). A junior sees magic; a mid-level engineer sees an implicit contract worth documenting; a senior asks whether a typed request-context object would be safer.

## Celery: functions that run elsewhere, later, maybe twice

Model: `@shared_task(queue=...)` registers a function; `.delay(args)` serializes args to Redis; a worker deserializes and runs. Everything follows from "elsewhere, later, maybe twice": pass **ids not objects** (`document_indexing_task.py#L33` takes `dataset_id`, `document_ids`), re-fetch and re-validate inside the task (`#L59-L91`), make retries safe, choose queues deliberately (`queue="dataset"` vs `"priority_dataset"`, `#L234-L259`).

## SSE vs WebSocket vs polling (this repo has all three lenses)

SSE for server→client streams (`ssePost`), Socket.IO for collaboration (`app_factory.py#L127-L138`, `ext_socketio`), and plain request/response elsewhere. Model to carry: SSE = one-directional, auto-reconnect-friendly, proxy-friendly; WebSocket = bidirectional, heavier operationally; polling = simplest, latency/cost tradeoff.

## Drills

1. Find where the workflow editor decides undo/redo history entries (`history-slice.ts`). What user actions create an entry? Rubric — Strong: you can explain why *not* every node drag should snapshot.
2. Convert (on paper) `useAppDetail` to the "raw useEffect" style, then list four bugs the Query version prevents (dedupe, races, cache, retries).
3. Write the Express equivalent of the `ChatMessageApi` decorator stack as middleware array, in order.

## Interview angle

1. "Why TanStack Query over fetch-in-useEffect?" → dedupe/staleness/invalidations with the delete-app example. (→ 08/02)
2. "When Zustand/Redux vs component state?" → chat tree vs workflow canvas contrast above. (→ 08/02)
3. "Explain middleware." → Flask decorator stack ↔ Express chain mapping. (→ 08/03)
4. "How do background jobs change your data-passing rules?" → ids-not-objects + revalidation. (→ 08/03, 08/04)
