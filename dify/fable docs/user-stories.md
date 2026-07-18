# User Stories — Upskilling Build Plans (Plans Only, Not Implemented)

Twenty user stories against the Dify codebase, each with a detailed implementation plan, designed to level a junior fullstack JS engineer toward mid-level. **Nothing here is implemented** — these are plans. Difficulty climbs roughly from US-1 to US-20. Anchors were verified against the working tree on 2026-07-10; conventions referenced: `api/AGENTS.md`, `web/AGENTS.md`. Companion curriculum: [`fable docs/upskill/`](upskill/README.md).

Per-story format: Story · Learning goals · Read first · Implementation plan · Tests · Risks & rollback · Definition of done.

---

## US-1: Sortable "Last updated" column on the apps list

**Story**: As a workspace member, I want to sort my apps by last update so I can find what I was working on.
**Learning goals**: full-stack field exposure; pagination/sort params; avoiding N+1.
**Read first**: `api/models/model.py#L421-L423` (`App.updated_at`), apps list controller in `api/controllers/console/app/app.py`, `web/service/use-apps.ts`, apps grid components under `web/app/components/apps/`.
**Plan**:
1. Confirm the list endpoint's existing query params (page/limit/filters) and response serializer; check whether `updated_at` is already returned.
2. Add a `sort_by` param (whitelist: `created_at`, `updated_at`, `name`) validated by the endpoint's Pydantic query model, defaulting to current behavior.
3. Apply ordering **on the column only** (never via lazy properties like `App.site` — N+1, see `model.py#L441-L443`).
4. Frontend: extend the query hook's key with the sort param (cache correctness), add a sort dropdown using an existing `dify-ui` select; strings via `web/i18n/en-US/`.
5. Persist the user's choice in URL params (`nuqs` pattern) rather than component state.
**Tests**: backend unit test for param validation + ordering; hook test asserting distinct query keys per sort; manual pagination sanity.
**Risks/rollback**: additive param, default unchanged — trivially revertible.
**Done when**: sort works across pages, no additional SQL queries per row (verify with SQL echo).

## US-2: Empty-query guard on chat endpoints

**Story**: As an API consumer, I get a clear 400 for whitespace-only queries instead of a wasted model invocation.
**Learning goals**: Pydantic validators; blast-radius analysis; cost-aware validation.
**Read first**: `api/controllers/console/app/completion.py#L43-L68`; sibling payloads in `controllers/service_api/app/` and `controllers/web/`; grep frontend senders (`chat/hooks.ts#L685-L692`).
**Plan**:
1. Inventory every chat/completion payload model across the three surfaces; note which share base classes.
2. Add a `field_validator("query")` that strips and rejects empty strings on chat payloads (completion apps allow empty query by design — note the default at `completion.py#L52`; investigate before touching).
3. Grep for internal callers that legitimately send empty queries (suggested questions, opening statements) — adjust scope accordingly.
4. Return the standard 400 shape; add an i18n-ready error message on the web client if it can surface it.
**Tests**: unit tests per payload model (valid, empty, whitespace); one controller test asserting 400 and error code.
**Risks**: breaking a hidden internal caller — mitigated by the grep step; rollback = remove validator.
**Done when**: all three surfaces behave identically and the saved-model-call rationale is documented in the validator docstring.

## US-3: Relative "processing for X minutes" indicator on knowledge documents

**Story**: As a knowledge admin, I can see how long a document has been processing so I can spot stuck ones.
**Learning goals**: reading status machines; serializer changes; dayjs/i18n UI work.
**Read first**: `api/tasks/document_indexing_task.py#L93-L106` (`processing_started_at` set), document list controller under `api/controllers/console/datasets/`, knowledge list UI under `web/app/components/datasets/`.
**Plan**:
1. Confirm `processing_started_at` is in the document list response; add to the serializer if absent (additive).
2. Frontend: for non-terminal statuses, render relative time (existing dayjs util) next to the status badge; tooltip with absolute timestamp.
3. Threshold styling: > 1h processing renders warning color (constant, not config, for now).
4. i18n keys for the new strings.
**Tests**: component test with fixed clock (fake timers) across status/duration matrix.
**Risks**: none server-side; UI-only rollback.
**Done when**: a stuck document is visually distinguishable within one screen.

## US-4: Toast deduplication in the fetch layer

**Story**: As a user with a flaky connection, I see one error toast per failure burst, not a stack.
**Learning goals**: cross-cutting HTTP-layer concerns; TTL cache in a closure; vitest.
**Read first**: `web/service/fetch.ts#L61-L79` (`afterResponseErrorCode`), `web/service/base.spec.ts` for test style.
**Plan**:
1. Add a module-level `Map<string, number>` of recent toast keys (`code+message`) with a ~3s TTL, consulted before `toast.error`.
2. Prune lazily on insert (no timers leaking).
3. Respect the existing `silent` option untouched; different messages must still both show.
4. Export a `_resetToastDedupe()` test hook or inject the clock.
**Tests**: two identical failures within TTL → one toast; different messages → two; after TTL → two.
**Risks**: over-suppression — keyed by code+message and short TTL to bound it.
**Done when**: burst scenario (kill API, click around) produces a single toast per distinct error.

## US-5: Regression tests pinning the login anti-enumeration property

**Story**: As a maintainer, the "identical error for unknown email vs wrong password" property is protected by tests.
**Learning goals**: security properties as tests; service-layer unit testing; AAA style.
**Read first**: `api/services/account_service.py#L214-L243`; neighboring tests in `api/tests/unit_tests/services/`.
**Plan**:
1. Fixtures: one existing account with known password; use existing account factories if present (search first).
2. Tests: (a) unknown email raises `AccountPasswordError` with the canonical message; (b) wrong password raises same type+message; (c) banned account raises `AccountLoginError` (different by design — document why in a comment); (d) pending → active transition on successful login (`#L237-L239`).
3. Add a short comment in the service linking the uniform-message rationale.
**Tests**: are the deliverable.
**Risks**: none — test-only.
**Done when**: `make test TARGET_TESTS=...` green and the property is grep-discoverable ("enumeration").

## US-6: Document-count and quota display on dataset creation

**Story**: As a knowledge admin on a limited plan, I can see my document quota usage before an upload fails at the limit.
**Learning goals**: feature-service/billing reading; defensive duplicated checks; UX for quota errors.
**Read first**: quota checks at `api/services/dataset_service.py#L1982-L2006` and task-side re-checks `api/tasks/document_indexing_task.py#L64-L91`; `services/feature_service.py`; upload UI under `web/app/components/datasets/create/`.
**Plan**:
1. Locate the existing features endpoint the console loads (the exemption comment at `api/app_factory.py#L20-L32` names `/console/api/features`) — confirm vector-space/doc quota fields are exposed.
2. Frontend: on the upload step, render `used/limit` from the features payload; disable the confirm button when the selected file count would exceed it, mirroring the server's rule (server remains authoritative).
3. Map the server's quota `ValueError` messages to a friendly i18n error state instead of a raw toast.
**Tests**: component tests for the three states (under, at, over limit); no backend change expected.
**Risks**: client/server rule drift — mitigate by keeping the client check advisory (button hint), server rejection still handled.
**Done when**: an over-quota upload is prevented client-side *and* the server path still shows a friendly error if raced.

## US-7: "Copy cURL" button on the API access page

**Story**: As an app developer, I can copy a working cURL example (with my app token placeholder and correct base URL) for the chat endpoint.
**Learning goals**: service-API contract reading; small UI feature end-to-end; docs-as-product.
**Read first**: `api/controllers/service_api/app/completion.py` (request shape), `App.api_base_url` (`model.py#L461-L464`), the develop/API page under `web/app/components/develop/`.
**Plan**:
1. Build the cURL string client-side from the app's base URL + mode-specific endpoint + minimal valid JSON body (streaming mode, `user` field included — required per `FetchUserArg` docs).
2. Use the existing `copy-to-clipboard` util + feedback toast; i18n all labels.
3. Vary the snippet by app mode (chat vs completion vs workflow) using existing mode constants.
**Tests**: snapshot tests of generated snippets per mode.
**Risks**: snippet drift if the API changes — put the snippet builder near the API-doc constants so reviews co-locate them.
**Done when**: pasting the copied command with a real token against a dev server returns SSE events.

## US-8: Cross-tenant isolation test suite for datasets

**Story**: As a maintainer, dataset/document endpoints have a reusable two-tenant test harness proving isolation.
**Learning goals**: the *most important invariant* in the repo; fixture engineering; parametrized tests.
**Read first**: `api/AGENTS.md` tenant-scoping rule; scoped-query shape `document_indexing_task.py#L82-L84`; existing controller tests under `api/tests/unit_tests/controllers/`.
**Plan**:
1. Build a `two_tenants` fixture: tenants A/B, one account each, one dataset+document each (reuse existing factories where present).
2. Parametrize over read endpoints (dataset detail, document list, segment list, hit-testing) asserting A's session cannot read B's resources (expect 404-or-403 — record which, per endpoint, and flag inconsistencies as findings).
3. Add one write-path case (rename B's dataset from A's session).
4. Document the harness so future endpoints add one line to be covered.
**Tests**: are the deliverable; keep them unit-level (mock only auth, use real query logic against the test DB pattern used by neighboring tests).
**Risks**: none — tests only; findings (if any) go to the risk register, not silent fixes.
**Done when**: the parametrized suite runs green and the 403/404 policy table it produced is committed alongside.

## US-9: Missing-invalidation audit and fix for app mutations

**Story**: As a console user, every app mutation (rename, copy, switch mode, site toggle) refreshes every list that shows it.
**Learning goals**: TanStack Query invalidation discipline; migration-seam debugging; systematic audits.
**Read first**: the dual-key pattern `web/service/use-apps.ts#L60-L81`; legacy fetchers `web/service/apps.ts#L19-L120`; Trace 5 in [upskill/04/02-trace-tables](upskill/04-code-reading-gym/02-trace-tables.md).
**Plan**:
1. Inventory: table of every app mutation → which keys it invalidates today (grep `invalidateQueries` + `consoleQuery.apps`).
2. Inventory: every reader (component/hook) → which key system it reads.
3. Matrix-join to find gaps; fix each by adding the missing invalidation in the mutation hook (or migrating the mutation to the typed client following `useDeleteAppMutation`'s shape).
4. Centralize: extract a `invalidateAppLists(queryClient)` helper so future mutations call one thing.
**Tests**: hook test per fixed mutation asserting both key systems invalidated.
**Risks**: over-invalidation is cheap; under is the bug — bias accordingly.
**Done when**: the matrix has no gaps and the helper exists.

## US-10: Structured logging pass for the indexing pipeline

**Story**: As an operator, every indexing failure log line carries tenant_id, dataset_id, and document_id.
**Learning goals**: observability house rules; log-context discipline; touching a pipeline safely.
**Read first**: logging rules in `api/AGENTS.md`; current logs in `document_indexing_task.py` (good: `#L113-L114`, `#L119`) and `indexing_runner.py#L58-L67` (error handler lacks tenant context).
**Plan**:
1. Audit every `logger.*` call in the two files + `tasks/` indexing siblings; list which ids each includes.
2. Add missing ids; standardize the format (`"... tenant=%s dataset=%s document=%s"`) — greppable, not JSON (match existing style).
3. Confirm level policy: retryable=warning, terminal=error (house rule); fix miscategorized lines.
4. No behavior changes — logging only.
**Tests**: caplog-based assertions on two representative paths (billing failure, runner exception).
**Risks**: none functional; noisy-log review risk — keep the diff mechanical.
**Done when**: a simulated failure produces logs from which you can identify the document without a DB query.

## US-11: Stale-indexing sweeper (scheduled task)

**Story**: As an operator, documents stuck in non-terminal indexing states for more than N hours are automatically marked ERROR with a clear message.
**Learning goals**: designing for liveness; scheduled tasks; idempotency; config plumbing.
**Read first**: state transitions (`document_indexing_task.py#L93-L106`, `indexing_runner.py#L58-L67`); `api/schedule/` for the scheduling pattern; `configs/` for adding `dify_config` settings; pause semantics (`DocumentIsPausedError`, `Document.is_paused`-like columns in `models/dataset.py#L487+` — verify exact fields first).
**Plan**:
1. Archaeology: confirm no existing sweeper (`rg -i "stale|sweep|stuck" api/schedule api/tasks`); confirm pause is distinguishable from crash (paused docs must be exempt).
2. Config: `INDEXING_STALE_THRESHOLD_HOURS` via `dify_config` + `.env.example` (dotenv-linter runs in `make lint`).
3. Task: select documents in non-terminal statuses with `processing_started_at < now - threshold`, not paused, batched (limit 500/run); set ERROR + message "Indexing timed out after Xh — retry from the document page"; log per house rules with ids.
4. Schedule it following an existing `schedule/` entry's registration pattern.
5. Idempotency: re-running is a no-op (terminal states excluded by the WHERE clause).
6. Ship dark first: dry-run mode (log matches, don't write) behind config, one release before enabling.
**Tests**: fake-clock unit tests: under-threshold untouched, over-threshold flipped, paused exempt, terminal exempt, batch limit respected, dry-run writes nothing.
**Risks**: falsely killing slow-but-alive jobs — threshold conservative (default 6-12h) + dry-run soak; rollback = disable schedule entry.
**Done when**: dry-run soak in dev shows only genuinely dead documents matched, then live mode flips them.

## US-12: Chat error item with retry (frontend)

**Story**: As a chat user, a failed answer shows an inline error state with a retry button that regenerates as a sibling of the same question.
**Learning goals**: optimistic-UI failure paths; message-tree semantics; hooks testing with mocked SSE.
**Read first**: placeholder lifecycle `web/app/components/base/chat/chat/hooks.ts#L620-L709`; `onError` handling in `ssePost` (`base.ts#L555-L566`); tree helpers (`updateCurrentQAOnTree`); test patterns `chat/__tests__/hooks.spec.tsx#L150-L320`.
**Plan**:
1. Map the current failure path: what happens to the placeholder answer on `onError` today (read carefully; document before changing).
2. Extend the answer item type with an `error` field (code+message) set in `onError`; keep the item in the tree (don't orphan the question).
3. UI: error card in the answer bubble (i18n), retry button calling `handleSend` with the *original question's* `parent_message_id` — a regenerate-sibling, consistent with existing branching.
4. Guard interplay: retry must respect `isRespondingRef`; disable while responding.
5. Ensure `errorCode` survives from SSE to the item (Trace 4 found this may be lossy — fix if needed).
**Tests**: hooks.spec-style: failing `ssePost` mock → error item present; retry produces a sibling; double-click guarded.
**Risks**: tree-state regressions — the tests are the mitigation; UI-only rollback.
**Done when**: kill the API mid-answer in dev; you get an inline error and a working retry, and the chat log's history remains consistent.

## US-13: Per-mutation optimistic rename for apps

**Story**: As a console user, renaming an app updates the UI instantly and rolls back on failure.
**Learning goals**: optimistic mutations with TanStack Query (`onMutate`/`onError`/`onSettled`); cache surgery.
**Read first**: `useDeleteAppMutation` shape (`use-apps.ts#L60-L81`); `updateAppInfo` (`service/apps.ts#L39-L60`); Query docs pattern for optimistic updates.
**Plan**:
1. Create `useRenameAppMutation`: `onMutate` cancels outgoing list/detail queries, snapshots previous data, writes the new name into both list and detail caches; `onError` restores snapshots; `onSettled` invalidates (both key systems).
2. Wire the rename modal to it; disable the submit while pending.
3. Keep server as source of truth: final invalidation always runs.
**Tests**: hook tests: success (optimistic value visible pre-response), failure (rollback), settled (invalidations fired).
**Risks**: cache-shape coupling (writing into cached list structures) — encapsulate the cache-write in one helper next to the key factory.
**Done when**: rename feels instant on a throttled network and a forced 500 visibly rolls back.

## US-14: Service API — segment listing pagination parity

**Story**: As a service-API consumer, segment listing supports the same pagination envelope as other list endpoints, documented in the OpenAPI spec.
**Learning goals**: public-contract discipline; keyset-vs-offset decision; schema registration (`controllers/API_SCHEMA_GUIDE.md`).
**Read first**: existing paginated service-API endpoints under `api/controllers/service_api/dataset/` (copy their envelope exactly); `validate_app_token` + dataset scoping (`service_api/wraps.py`); `DocumentSegment` (`models/dataset.py#L833`).
**Plan**:
1. Contract inventory: document the current response of the segments endpoint (if paginated already, this story becomes "verify + spec"; check first).
2. Match the repo's existing pagination convention (likely page/limit + total) — parity beats novelty on a public surface.
3. Pydantic query model with bounds (limit ≤ 100), response DTO registered per the schema guide; serialize via `model_validate(...).model_dump(...)`.
4. Query: scoped by tenant → dataset → document chain, ordered by position; index check with EXPLAIN (position or created_at).
5. Update `sdks/nodejs-client` typings if the SDK exposes this endpoint.
**Tests**: pagination bounds, tenant isolation (US-8 harness), ordering stability, empty page.
**Risks**: breaking existing consumers if the endpoint exists unpaginated — additive params + default = old behavior; rollback safe.
**Done when**: spec, SDK, and endpoint agree, proven by a contract test.

## US-15: Dataset permission management UI (partial members)

**Story**: As a dataset owner, I can grant access to specific workspace members, matching the `partial_members` permission model.
**Learning goals**: authorization end-to-end; ACL data model; member-picker UI; authz *for changing* authz.
**Read first**: `DatasetPermission` / `DatasetPermissionEnum` (`models/dataset.py#L1355`, `#L160`); dataset settings services in `services/dataset_service.py` (search `DatasetPermission` usages — backend support likely exists); members endpoint under `controllers/console/workspace/`; dataset settings UI under `web/app/components/datasets/settings/`.
**Plan**:
1. Backend archaeology: enumerate existing endpoints for reading/writing dataset permissions; the story may be UI-plus-gaps rather than greenfield. Produce a capability table first.
2. Fill any backend gaps: endpoints to set permission mode + member list, guarded by dataset-editor/admin checks; validate member ids belong to the same tenant (isolation!).
3. UI: permission mode radio (only me / all team / partial) + member multi-select (existing member-picker component if present — search before building); optimistic off, plain invalidation on.
4. Enforcement audit: verify list/detail/hit-testing endpoints actually filter by `DatasetPermission` for non-privileged roles — write the tests that prove it (extends US-8).
**Tests**: permission matrix tests (owner/admin/editor/normal × three modes); cross-tenant member id rejection; UI component tests for the three modes.
**Risks**: enforcement gaps discovered mid-story — timebox: report as findings, fix only the write-path in this story.
**Done when**: a `normal`-role member sees exactly the datasets the matrix says they should.

## US-16: Workspace "last active" honesty feature

**Story**: As a workspace owner, the members page shows meaningful activity recency so I can manage seats.
**Learning goals**: data-honesty judgment; finding the writer of a column before exposing it.
**Read first**: `Account.last_active_at` (`models/account.py#L104-L106`); find every writer (`rg "last_active_at" api/`); members list controller under `controllers/console/workspace/`.
**Plan**:
1. Archaeology first: determine when `last_active_at` is actually updated (login only? per request? never?). The finding decides the story:
   - Updated per-request → expose as "Last active".
   - Login-only → expose as "Last login" (there's also `last_login_at`, `#L102`) and *label honestly*.
   - Effectively never → story becomes: update it cheaply (e.g., throttled touch — at most once/hour per account via Redis flag in `login_required`'s path) then expose. Get the throttle reviewed carefully; hot-path writes are Kata-6 territory.
2. Serializer + UI column with relative time; sortable per US-1's pattern.
3. Privacy note: visible to admin/owner only — decorator check on the endpoint if not already privileged.
**Tests**: writer-path unit test (throttle: two requests in an hour → one write); serializer includes the field for admins, not for normal members (verify current behavior first).
**Risks**: per-request DB writes — the throttle is the mitigation; rollback = stop writing, column stays.
**Done when**: the displayed label matches what the data actually means, verified against the writer inventory.

## US-17: Signed webhook on workflow completion (design-first story)

**Story**: As an app developer, my server receives a signed webhook when a workflow run completes, so I don't poll.
**Learning goals**: outbound-callback design: HMAC signing, retries, SSRF discipline, secret storage; RFC writing.
**Read first**: task/queue patterns (`tasks/`, `document_indexing_task.py`); ssrf rule (`api/AGENTS.md`; proxy service in `docker-compose.middleware.yaml` L197); provider-credential encryption patterns (`models/provider.py`); workflow completion event location (`core/app/apps/workflow/` generators / task pipeline — find where terminal state is known).
**Plan**:
1. **RFC first** (template in [upskill/07/02](upskill/07-career-and-collaboration/02-writing-prs-and-rfcs.md)): payload envelope (versioned `v1`: run id, app id, status, outputs summary — no raw tenant data by default), signature (`X-Signature: hmac-sha256(ts + body)`), replay window, retry policy (3 attempts, exponential, dead-letter status).
2. Data model: `app_webhooks` table (id, tenant_id, app_id, url, secret_encrypted, enabled, created_by) + delivery-attempt log table; migration additive.
3. Delivery task in `tasks/` on a dedicated queue; **all egress via the ssrf_proxy helper**; timeouts mandatory.
4. Trigger point: hook into the terminal-state code path found in step 0 — smallest possible touch; if a workflow-events abstraction exists, subscribe there rather than editing the engine.
5. Management endpoints (console surface): CRUD + "send test event," admin/editor-gated; settings UI panel.
6. Secret rotation: two active secrets, dual-sign during rotation window.
**Tests**: signature vectors (known key/body → known sig); retry schedule with fake clock; SSRF (private-IP URL rejected); tenant isolation on CRUD; end-to-end with a local receiver in dev.
**Risks**: this touches the engine's completion path — keep the hook one line + task dispatch; global kill-switch config; rollback = disable flag.
**Done when**: RFC reviewed; dev demo: workflow completes → local server verifies signature and prints payload.

## US-18: Rate-limit response headers + archaeology on `api_rpm`/`api_rph`

**Story**: As a service-API consumer, I can see my concurrency limits in response headers, and the platform's documented limits match reality.
**Learning goals**: archaeology as engineering; middleware/header design; honest docs.
**Read first**: `App.api_rpm/api_rph` (`models/model.py#L411-L412`); `RateLimit` (`core/app/features/rate_limiting/`); `_get_max_active_requests` (`app_generate_service.py#L272-L291`); service-API wraps.
**Plan**:
1. **Phase 1 (findings memo)**: trace whether `api_rpm`/`api_rph` are enforced anywhere (`rg "api_rpm|api_rph" api/`). Deliver a one-page memo: enforced / vestigial / partially wired, with anchors. This memo is the story's core value — do not skip to code.
2. **Phase 2 (headers)**: expose what is *actually* enforced: active-request limit as `X-Concurrency-Limit` + `X-Concurrency-Remaining` (semantics: concurrent, not windowed — name them honestly, don't fake `X-RateLimit-Reset`). Emit from the generation endpoints' response path where `rate_limit` state is available.
3. **Phase 3 (decision)**: if rpm/rph are vestigial, propose either enforcement (bigger story, separate RFC) or column deprecation note — maintainer's call; you present options.
**Tests**: header presence/values under concurrent requests (unit-level with mocked RateLimit); memo reviewed.
**Risks**: header names become contract — prefix experimental (`X-Dify-…`) until blessed.
**Done when**: memo answers the enforcement question with evidence, and headers reflect only true limits.

## US-19: Frontend typed-client migration — one full resource (tags)

**Story**: As a maintainer, the tags resource uses the generated typed client and key factories end-to-end, with legacy fetchers deleted.
**Learning goals**: executing a live migration slice; codegen workflow; stale-cache prevention.
**Read first**: migration exemplar `use-apps.ts#L51-L81`; `web/service/client.ts` / `consoleQuery` factories; legacy tag service (`web/service/tag.ts` or within `common.ts` — locate first); tag consumers (`rg -l "tag" web/app/components --type=tsx` narrowed by import).
**Plan**:
1. Inventory: every legacy tag fetcher export and every component importing it; every query key referencing tags.
2. Verify/extend the typed client covers all tag endpoints (backend OpenAPI registration may need additions — follow `controllers/API_SCHEMA_GUIDE.md` if so; that's part of the slice).
3. Introduce `useTagsQuery`/`useCreateTagMutation`/etc. with factory keys; migrate consumers one PR-reviewable chunk at a time.
4. Invalidation parity: mutations invalidate the new keys; grep for any *other* resource whose cache embeds tags (apps with tag filters!) and invalidate those too.
5. Delete legacy exports last; `knip` confirms no dead code remains.
**Tests**: hook tests per mutation (invalidations); one integration-style component test for the tag filter flow surviving the swap.
**Risks**: hidden readers of legacy keys → the inventory step; rollback = revert the final deletion PR only (earlier PRs are additive).
**Done when**: `rg` finds zero legacy tag fetcher imports and the tag-filter UI works with devtools showing only factory keys.

## US-20: Indexing operations mini-dashboard (console admin)

**Story**: As a workspace admin, I can see my workspace's indexing health at a glance: documents per status, oldest in-progress age, and queue backlog.
**Learning goals**: read-only feature spanning aggregates, Redis introspection, charts; performance-aware SQL; capstone integration of prior stories.
**Read first**: status fields (`models/dataset.py#L487+`); tenant queue (`core/rag/pipeline/queue.py` + `document_indexing_task.py#L192-L231`); an existing console statistics endpoint for the response pattern (`use-apps.ts#L83-L89` shows the frontend shape); `echarts` usage elsewhere in the console for chart conventions.
**Plan**:
1. Backend: one endpoint `GET /console/api/datasets/indexing-overview` returning: counts by `indexing_status` (grouped SQL, tenant-scoped — a single `GROUP BY` over an indexed column set; run EXPLAIN with realistic volume), oldest non-terminal `processing_started_at` age, and this tenant's isolated-queue depth (expose a `size()` on `TenantIsolatedTaskQueue` if missing — read its Redis structure first).
2. Guard with dataset-editor-or-admin decorator; document the choice.
3. Cache the aggregate for 30s in Redis (dashboard, not control panel) — key includes tenant id.
4. Frontend: a small overview card section on the knowledge landing page: status donut (echarts, following existing chart component conventions), stat tiles for age and backlog, warning styling when age > US-11's threshold (link the two features conceptually in the UI copy).
5. Auto-refresh via `refetchInterval` 30s **only while the tab is visible** (Query's `refetchIntervalInBackground: false` default — deliberate contrast with review-kata 7's lesson).
**Tests**: aggregate endpoint unit tests (fixtures across statuses/tenants — isolation!), cache behavior, queue-depth fake; component tests for the three visual states (healthy / stuck warning / empty).
**Risks**: aggregate query cost on huge tables — the 30s cache plus indexed columns bound it; rollback = feature-flag the card.
**Done when**: with US-11's sweeper disabled and a killed worker, the dashboard visibly shows the stuck document age climbing — the two stories together turn a silent failure mode into an observable, self-healing one.

---

## Suggested sequencing

| Phase | Stories | Theme |
| --- | --- | --- |
| Warm-up (week 1-2) | US-1..US-5 | one layer at a time, tests always |
| Cross-layer (week 3-5) | US-6..US-10 | contracts, audits, observability |
| Reliability & authz (week 6-9) | US-11, US-12, US-13, US-15, US-16 | failure paths and permissions |
| Contracts & migration (week 10-12) | US-14, US-18, US-19 | public surfaces, archaeology, migration |
| Capstones | US-17, US-20 | design-first + integrative build |

Each completed story should immediately produce a STAR worksheet entry ([upskill/08-interview-prep/06-behavioral-star-stories.md](upskill/08-interview-prep/06-behavioral-star-stories.md)).
