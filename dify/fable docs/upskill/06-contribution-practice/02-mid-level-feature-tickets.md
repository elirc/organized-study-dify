# Mid-Level Feature Tickets

Ten cross-layer tickets. **Rule: write a half-page design note before coding** (data model → API → UI → tests → risks → rollback). Each ticket lists the layers it crosses and its rollback story — practicing rollback thinking is the point.

---

## M1: Archive apps (soft delete with restore)
Medium · 2-3d · Layers: schema + API + UI + tests
Story: As a workspace admin, I can archive an app (hidden from lists, restorable) instead of deleting it.
Design questions to answer first: new `status` value (`AppStatus` exists — `model.py#L406-L408`) vs new `archived_at` column? What happens to its published site and service API (`enable_site/enable_api`) while archived — `validate_app_token` already rejects non-`normal` status (`service_api/wraps.py#L67-L68`): is that the behavior you want?
Plan: migration (additive) → service methods with role check (admin/owner) → list filters → UI menu + archived tab → tests: cross-tenant, role-denied, service-API-rejected-while-archived.
Risk/rollback: filter changes touch every list query; rollback = feature flag the UI, statuses are inert data.
Interview story potential: full-stack lifecycle feature with an authz and public-API interaction — a complete system-design-in-miniature.

## M2: Per-document indexing progress (segments done / total)
Medium-hard · 3d · Layers: task + API + UI
Story: As a knowledge admin, I see percentage progress during indexing, not just a status word.
Anchors: runner loop (`indexing_runner.py#L69-L119`), status columns, knowledge UI list.
Design: where to store progress (columns vs Redis)? update frequency (every segment = write amplification; batch every N)?
Tests: progress monotonicity; ERROR preserves last progress.
Rollback: additive columns; UI behind flag.
Interview story: "I added progress reporting to a long pipeline and reasoned about write amplification."

## M3: Stale-indexing sweeper
Medium · 2d · Layers: schedule + task + tests (pairs with T8)
Story: As an operator, documents stuck in non-terminal states > N hours are marked ERROR with an explanatory message.
Anchors: state machine (Trace 2), `api/schedule/` for the cron pattern, `document_indexing_task.py#L93-L106`.
Design: threshold config via `dify_config`; idempotency (sweeping twice is harmless); exclusion of legitimately-paused docs.
Interview story: closing a reliability gap you *identified from reading* — the strongest kind of story.

## M4: Service-API endpoint: `GET /v1/datasets/<id>/documents/<doc_id>/segments` pagination parity
Medium · 2d · Layers: API + SDK + docs
Story: As an API consumer, segment listing supports the same pagination contract as other list endpoints.
Anchors: `controllers/service_api/dataset/`, existing paginated endpoints for the pattern, `sdks/nodejs-client`.
Design: keyset vs offset (match existing convention); response envelope consistency.
Tests: tenant isolation, dataset-permission enforcement, page bounds.
Interview story: public-API design under backward-compatibility constraints.

## M5: Chat error states with retry affordance
Medium · 2-3d · Layers: frontend only, but deep
Story: As a chat user, a failed answer shows an inline error with a "retry" that reuses the same parent message (sibling regenerate).
Anchors: placeholder lifecycle (`chat/hooks.ts#L644-L679`), onError path, tree branching via `parent_message_id`.
Design: error item shape in the tree; retry = `handleSend` with same parent; dedupe guard interplay (`isRespondingRef`).
Tests: hooks.spec pattern with failing `ssePost` mock (`__tests__/hooks.spec.tsx`).
Interview story: optimistic-UI failure handling — the exact question interviewers ask about optimistic updates.

## M6: Workspace member "last active" column
Medium · 2d · Layers: schema(read) + API + UI
Story: As an owner, the members page shows each member's `last_active_at` so I can prune seats.
Anchors: `Account.last_active_at` (`account.py#L104-L106`), members controller in `controllers/console/workspace/`.
Design: is `last_active_at` updated often enough to be truthful? (find the updater; if it's login-only, rename the UI label honestly).
Interview story: data-honesty judgment — "I checked what the column actually means before exposing it."

## M7: Rate-limit headers on service API
Medium-hard · 3d · Layers: API middleware + docs
Story: As an API consumer, responses include `X-RateLimit-*` style headers reflecting the app-level active-request limit.
Anchors: `RateLimit` (`core/app/features/rate_limiting/`), `_get_max_active_requests` (`app_generate_service.py#L272-L291`), `api_rpm/api_rph` columns (`model.py#L411-L412` — investigate whether enforced!).
Design: which limits are real vs aspirational; header semantics for concurrent (not windowed) limits.
Interview story: you discovered whether `api_rpm/api_rph` are actually enforced — genuine archaeology with a customer-facing result.

## M8: Dataset permission UI for `partial_members`
Medium-hard · 3-4d · Layers: API + UI + tests
Story: As a dataset owner, I can grant specific members access via a member-picker, matching the `DatasetPermissionEnum.PARTIAL_TEAM` model.
Anchors: `DatasetPermission` (`dataset.py#L1355`), `DatasetPermissionEnum` (L160), dataset settings UI.
Design: check what backend support already exists (service methods) vs UI-only gap; authorization for *changing* permissions (owner/admin only?).
Tests: cross-tenant + non-member rejection; permission-change audit log?
Interview story: authorization feature end-to-end — top-tier interview material.

## M9: Export conversation as markdown (console)
Medium · 2-3d · Layers: API + UI
Story: As an app editor, I can export a conversation thread as markdown for sharing.
Anchors: `Conversation`/`Message` (`model.py#L1044`, `L1393`); Kata 1's lessons (streaming, scoping, authz) apply verbatim.
Design: server-side render vs client-side from loaded messages (client-side is smaller blast radius — justify!); tree → linear thread selection.
Interview story: you applied review-kata lessons to your own feature — tell it that way.

## M10: Frontend typed-client migration for one resource
Medium · 2d · Layers: frontend service + tests
Story: As a maintainer, resource X (pick tags or workspaces) uses `consoleClient`/key factories instead of hand-rolled fetchers, with legacy keys removed.
Anchors: migration pattern in `use-apps.ts#L51-L81`, `service/client.ts`.
Design: inventory all readers of the legacy keys first (the Trace 5 lesson); delete legacy keys only after all readers moved.
Interview story: "I executed one slice of a live migration without a stale-cache regression" — process maturity evidence.

---

**Design-note template** (fill before coding, attach to your PR): Problem · Non-goals · Data model delta · API delta · UI delta · AuthZ decision · Test list · Failure modes · Rollback · Open questions.
