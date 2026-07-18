# Review Katas

Eight fake PRs. For each: read the intent and diff summary, review it *in writing* (blocking / important / optional findings), then compare with the expected findings. Findings language matters — kind, specific, actionable. Time-box: 20 minutes each.

Grading — you pass a kata when you catch all Blocking findings; Solid adds most Important; Strong adds an Optional insight the answer key missed *and* your comments read like a colleague, not a linter.

---

## Kata 1: "Add `GET /console/api/apps/<id>/export-conversations`"

Author intent: let editors download all conversations as CSV.
Fake diff summary: new Resource in `controllers/console/app/`; queries `Message` by `app_id` only; builds CSV in memory; `@login_required` only.
Files this resembles: `api/controllers/console/app/completion.py#L146-L160` (decorator stack), `api/models/model.py#L1393` (Message).
Expected findings — **Blocking**: missing tenant/app scoping via `get_app_model` (IDOR); missing `account_initialization_required` + role check (data export deserves ≥ editor). **Important**: unbounded query + in-memory CSV → OOM on big apps (paginate/stream); no audit logging for bulk export. **Optional**: CSV injection (`=cmd`) escaping; filename i18n.
Good review comment example:
> Loading every message into memory will OOM on large apps — `Message` has no row cap here. Could we stream with `yield` + server-side cursor, like the existing export patterns? Also needs `@get_app_model` so the app is tenant-scoped; bare `app_id` lookups are how cross-tenant reads slip in.

## Kata 2: "Speed up app list by caching in module scope"

Intent: cache `fetch_app_list` results in a module-level dict keyed by tenant.
Resembles: services layer; `extensions/ext_redis.py` exists for real caching.
Expected — **Blocking**: module-level dict is per-process, never invalidated, survives across requests (stale + memory growth), and breaks multi-worker deployments. **Important**: no TTL/invalidations on app create/delete; use Redis with explicit keys or TanStack Query on the client (already caches — `use-apps.ts`). **Optional**: measure first — is the list actually slow, or is it the `App.site` N+1 (`model.py#L441-L443`)?

## Kata 3: "Retry LLM calls 5 times"

Intent: wrap model invocation in `for attempt in range(5)` with `time.sleep(2**attempt)`.
Resembles: `core/model_manager.py` invocation paths; error taxonomy in `completion.py#L186-L196`.
Expected — **Blocking**: retrying non-idempotent/user-billed generation on *all* exceptions (quota errors, auth errors must not retry); synchronous `sleep` holds a gunicorn worker up to 62s. **Important**: retry belongs at a layer that can distinguish `InvokeError` subtypes; jitter missing (thundering herd). **Optional**: config-driven attempts; per-provider policies.

## Kata 4: "Show document error details in the UI"

Intent: render `document.error` raw into the knowledge UI with `dangerouslySetInnerHTML` to preserve stack-trace line breaks.
Resembles: `Document.error` written at `indexing_runner.py#L63-L66`; frontend knowledge components.
Expected — **Blocking**: XSS — error strings can contain user-influenced content (filenames, URLs); use text nodes + `white-space: pre-wrap`. **Important**: hardcoded English label instead of `web/i18n/en-US/` (repo rule); consider truncation. **Optional**: map common errors to friendly i18n messages.

## Kata 5: "Refactor: move billing checks out of the indexing task"

Intent: "checks already ran in the service; delete the duplicate ones in `_document_indexing` (`document_indexing_task.py#L64-L91`)."
Expected — **Blocking**: the queue is a trust boundary — plan can downgrade, quota can fill between enqueue and execution; the duplication is intentional defense (state the *why*, don't just revert). **Important**: if dedupe is truly wanted, extract a shared `check_upload_quota(tenant)` used by both, keeping the task-side call. **Optional**: docstring documenting the boundary reasoning so the next person doesn't re-propose this.

## Kata 6: "Add `use_count` to ApiToken and increment on every request"

Intent: `api_token.use_count += 1; db.session.commit()` inside `validate_app_token`.
Resembles: `service_api/wraps.py#L55-L120`; note `record_token_usage` already exists (`#L23`).
Expected — **Blocking**: a synchronous write + commit on *every* API request on the hot path (latency + row contention on one row per app — lock convoy). **Important**: read-modify-write races undercount; use the existing usage-recording path / Redis counter flushed periodically. **Optional**: does product actually need per-request counts vs last_used_at?

## Kata 7: "Frontend: refetch app list every 5 seconds"

Intent: `useQuery({..., refetchInterval: 5000})` on the apps list so new apps appear.
Expected — **Important**: polling every 5s for a rarely-changing list wastes battery/server; invalidation-on-mutation already covers same-tab changes (`use-apps.ts#L70-L78`); consider `refetchOnWindowFocus` (default) for cross-tab. **Optional**: if real-time is a requirement, that's Socket.IO territory (`ext_socketio`) — scope question for product. *(Note: nothing here is Blocking — recognizing severity correctly is part of the kata.)*

## Kata 8: "Add DELETE /datasets/<id> cascade in the request"

Intent: delete dataset + all documents + segments + vector collection synchronously in the controller.
Resembles: existing `clean_dataset_task.py` (async cleanup) and dataset controllers.
Expected — **Blocking**: unbounded synchronous cascade in a web request (timeouts, partial deletes with no resume); vector-store call can hang → worker starvation. **Important**: repo already has the pattern — mark row deleted/disabled synchronously, enqueue idempotent cleanup task; follow it. **Optional**: soft-delete window for undo; authorization = admin/owner or dataset permission check, not just editor.

---

**Meta-drill**: after finishing all eight, write your personal review checklist (max 10 lines) ordered by what you actually missed most. Compare with [07-career-and-collaboration/01-code-review-mindset.md](../07-career-and-collaboration/01-code-review-mindset.md).
