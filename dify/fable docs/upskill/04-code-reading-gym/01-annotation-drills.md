# Annotation Drills

For each excerpt (open the real file — don't work from memory): annotate **inputs**, **outputs**, **dependencies**, **invariants**, **side effects**, **failure modes**. Then check against the notes. 15–20 minutes each.

**Self-grading rubric (applies to all drills)** — Basic: correct inputs/outputs. Solid: names at least one invariant and every side effect, including "hidden" ones (DB write, Redis, global mutation). Strong: also names a realistic failure mode *not* handled by the code, and what test would catch it.

---

### Drill 1 — `AppGenerateService.generate` (`api/services/app_generate_service.py#L87-L270`)

Notes to check yourself against: inputs include a *union-typed* user (Account|EndUser); output is either a Mapping (blocking) or an SSE generator. Side effects: quota reserve/commit/refund (Redis/billing), rate-limit enter/exit, Celery enqueue (streaming advanced/workflow modes only). Invariants: `request_id` uniquely pairs enter/exit; `workflow_run_id` minted before enqueue. Unhandled-ish failure: generator abandoned by client — who exits the rate limit? (Trace `rate_limit.generate`.)

### Drill 2 — `login_required` (`api/libs/login.py#L51-L102`)

Check: EXEMPT_METHODS (OPTIONS) and `LOGIN_DISABLED` bypass both authn *and* CSRF — is that safe in tests only? Side effect: `g._login_user` assignment. Invariant: CSRF runs only for authenticated non-exempt requests. Failure mode: a view mounted without this decorator has *no* CSRF check at all — decorator-based security is opt-in per route.

### Drill 3 — `_document_indexing` (`api/tasks/document_indexing_task.py#L49-L190`)

Check: three separate session lifetimes (limit-check, phase-1 status, post-success summary). Invariant: documents always reach a terminal or progress status even on billing failure (`#L81-L91`). Side effects: chained `generate_summary_index_task.delay`. Failure: dataset deleted mid-run → `ObjectDeletedError` handled in runner (`indexing_runner.py#L124-L125`) — what about deleted *between* sessions here?

### Drill 4 — `validate_app_token` (`api/controllers/service_api/wraps.py#L55-L120`)

Check: dual overloads for decorator-with/without-args; kwargs injection contract; the owner-login fallback's privilege implications; `request.get_json()` called inside the decorator (consumes/parses body — interaction with the view's own parsing?).

### Drill 5 — `ssePost` (`web/service/base.ts#L452-L580`)

Check: inputs include ~30 optional callbacks; output is `void` — all results flow through callbacks (CPS style). Side effects: global toasts, redirects (`requiredWebSSOLogin`), recursive self-call on 401. Invariant: exactly one of onError/onCompleted terminal call per stream — verify by reading `handleStream`. Failure: non-JSON error body → `res.json()` rejection unhandled in the non-401 branch (`#L548-L551`)?

### Drill 6 — `handleSend` (`web/app/components/base/chat/chat/hooks.ts#L620-L709`)

Check: guard `isRespondingRef` (why a ref, not state?); placeholder id scheme (`question-${Date.now()}`); which state updates happen synchronously before the request vs in callbacks; dependency array (`#L1170-L1181`) — find one dep whose change silently recreates the callback mid-conversation.

### Drill 7 — `Account.current_tenant` setter (`api/models/account.py#L129-L150`)

Check: opens its own Session with `expire_on_commit=False` (the comment explains DetachedInstanceError); mutates `self.role` as a side channel; failure mode: tenant join missing → silently sets `_current_tenant = None` (no raise) — every caller must handle a None tenant or crash later at `current_account_with_tenant`'s assert (`libs/login.py#L47`).

### Drill 8 — `create_flask_app_with_configs` before_request (`api/app_factory.py#L59-L96`)

Check: license verification failure modes (network error → force logout — availability coupling); exemption prefix list and *why* each entry exists (documented `#L20-L32`); string-prefix matching risks (`/console/api/features-x` also exempt?).

### Drill 9 — `WorkflowEntry.__init__` (`api/core/workflow/workflow_entry.py#L139-L220`)

Check: constructor raises on call-depth — validation in a constructor, pros/cons; layer registration order; which config values bound the engine; side effects: none until `run()` — good separation of assembly from execution.

### Drill 10 — `base()` fetch wrapper (`web/service/fetch.ts#L137-L249`)

Check: three API prefixes; ky `retry: {methods: []}` disables retries — why is that correct for a mutating client (idempotency!); content-type sniffing for blob returns; failure: `HTTPError` re-thrown as a synthetic `Response` object (`#L232-L235`) — callers must handle *both* thrown Responses and parse errors.
