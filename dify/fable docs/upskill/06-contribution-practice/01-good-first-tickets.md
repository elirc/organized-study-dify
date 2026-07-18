# Good First Tickets

Sixteen junior tickets. Common fields compressed: each lists Difficulty/Time, Story, Why good, Anchors, Plan, Risks/Checks, and Interview story potential. Acceptance criteria = the Story is true, existing tests pass, new test(s) added where noted. Before starting any of them, read the anchors and the *neighboring test file*.

---

## T1: Type the `is_agent` compatibility read (docs-first ticket)
Easy · 2-3h · Skills: reading, docstrings, invariants
Story: As a maintainer, I want `App.is_agent`'s mutation behavior documented in its docstring so nobody is surprised by a commit inside a property.
Why good: zero behavior change; huge review-learning density; follows `api/AGENTS.md` docstring policy.
Anchors: `api/models/model.py#L471-L484`.
Plan: write a class-level + property docstring stating the lazy-migration side effect and its trigger conditions.
Rejection risk: docstring that editorializes instead of stating the contract.
Interview story: "I documented a hidden write on a read path and proposed the follow-up fix."

## T2: Uniform UUID validation on `parent_message_id` everywhere
Easy · 3-4h
Story: As an API consumer, I get a 400 (not a 500) for malformed UUIDs on all message endpoints.
Anchors: pattern at `completion.py#L60-L65`; audit siblings in `controllers/console/app/` and `controllers/service_api/app/`.
Plan: grep payload models missing the validator; add `uuid_value` field validators + one unit test per model.
Checks: error shape consistent with existing 400s.
Interview story: "I hardened input validation across an API surface by extracting a repo-wide pattern."

## T3: Add `Last-Modified`-style `updated_at` to an app list response (if absent)
Easy · 3h
Story: As a console user, I can sort apps by last update.
Anchors: `App.updated_at` (`model.py#L421-L423`); list controller in `controllers/console/app/app.py`; frontend `web/service/use-apps.ts` types.
Plan: confirm serializer includes it; expose sort param following existing pagination params; wire dropdown in the apps page.
Risk: N+1 if the sort touches lazy properties — sort on the column only.
Interview story: full-stack field exposure with pagination discipline.

## T4: i18n sweep for one hardcoded string
Easy · 1-2h
Story: As a non-English user, I see translated text where a literal was hardcoded.
Anchors: rule in `web/AGENTS.md`; `web/i18n/en-US/`.
Plan: find one hardcoded string in a component (`rg '"[A-Z][a-z].*"' web/app/components | grep -v spec` and judgment); move to i18n keys; add to en-US only (other locales fall back).
Interview story: small, but demonstrates convention-following — pair it with a bigger one.

## T5: Toast dedupe for repeated fetch errors
Medium-easy · 4-6h
Story: As a user with a dropped connection, I get one error toast, not a stack of identical ones.
Anchors: `web/service/fetch.ts#L61-L79`, toast import from `@langgenius/dify-ui/toast`.
Plan: keep a short-TTL map of recent messages in the hook factory; skip duplicates; vitest with two rapid failures.
Risk: don't suppress *different* errors; key by code+message.
Interview story: UX polish via a cross-cutting layer — shows you know where the seam is.

## T6: Add missing `enabled` guard test for `useAppDetail`
Easy · 2h
Story: As a developer, the `enabled: !!appID` behavior is regression-tested.
Anchors: `use-apps.ts#L43-L49`; neighboring `web/service/__tests__/`.
Plan: renderHook without id → queryFn not called; with id → called once.
Interview story: "I added the test that documents a subtle correctness guard."

## T7: Log tenant + dataset ids in one indexing error path that lacks them
Easy · 2-3h
Story: As an operator, indexing failures are attributable without a DB query.
Anchors: house rule (`api/AGENTS.md` logging); compare `document_indexing_task.py#L119` (has dataset_id) vs sparse spots in `indexing_runner.py` error paths.
Plan: add ids to `_handle_indexing_error` log context (`indexing_runner.py#L58-L67`).
Interview story: observability micro-improvement — good "operations mindset" evidence.

## T8: Surface `processing_started_at` age in knowledge-document list UI
Medium · 1d
Story: As a knowledge admin, I can see how long a document has been processing, so stuck docs are visible.
Anchors: column set at `document_indexing_task.py#L104`; knowledge list components under `web/app/components/datasets/`.
Plan: confirm API returns the field (add to serializer if not); render relative time with existing dayjs util; i18n.
Interview story: turned an invisible failure mode into a visible one — pairs with the sweeper project (S2).

## T9: Escape-hatch audit: one `any` removal in `web/service`
Easy · 2-4h
Story: As a maintainer, one fewer `any` weakens the suppression ledger.
Anchors: `AGENTS.md` no-any rule; grep `: any` in `web/service/`.
Plan: pick one; type it properly (often a response type in `web/models/`); `pnpm type-check`.
Interview story: incremental typing improvement with zero behavior change.

## T10: 404 vs 403 consistency check for cross-tenant app access
Medium · 1d
Story: As a security-conscious maintainer, requesting another tenant's app id returns the same status as a nonexistent id.
Anchors: `get_app_model` (controllers/console/app/wraps.py); enumeration row in [05/05-security-checklist](../05-quality-engineering/05-security-checklist.md).
Plan: write the test first (two tenants fixture); if statuses differ, align to 404; document the policy in the wrap's docstring.
Interview story: a real security-behavior test — strong interview material.

## T11: `make test` docs fix for Windows contributors
Easy · 1-2h
Story: As a Windows contributor, the README tells me the working `TARGET_TESTS` form (path separators, quoting).
Anchors: `Makefile#L97-L105`; `api/README.md`.
Plan: verify actual behavior on your machine; add a short "Windows notes" section.
Interview story: small, but "I improved contributor onboarding docs after hitting the papercut" is a likable story.

## T12: Add missing invalidation for an app mutation
Medium · 0.5-1d
Story: As a console user, renaming/copying an app updates every list without refresh.
Anchors: `use-apps.ts#L60-L81` (the double-invalidate pattern); audit `updateAppInfo`/`copyApp` (`service/apps.ts#L39-L81`) call sites.
Plan: find a mutation missing one of the two key systems (Trace 5 drill found candidates); add invalidation + hook test.
Interview story: cache-consistency bug found by systematic audit — excellent for "tell me about a bug you found."

## T13: Reject empty `query` early in chat payloads
Easy · 3h
Story: As an API consumer, an empty/whitespace query gets a clear 400 instead of burning a model call.
Anchors: `ChatMessagePayload.query` (`completion.py#L55-L57`).
Plan: `field_validator` stripping/rejecting whitespace-only; confirm the web UI already prevents this (it should — verify) so blast radius is API-only; unit test.
Risk: check downstream features (suggested questions?) that might legitimately send empty queries — grep first.
Interview story: defensive validation with a cost argument (saved model invocations).

## T14: Storybook story for one dify-ui primitive
Easy · 3-4h
Story: As a designer/dev, primitive X has a story demonstrating its states.
Anchors: `packages/dify-ui/`, storybook deps in catalog.
Plan: copy an existing story's structure; cover disabled/loading/error states.
Interview story: design-system contribution; pairs with the overlay-contract reading.

## T15: Add a `--dry-run` flag to one maintenance command
Medium · 1d
Story: As an operator, I can preview what `commands/…` cleanup would delete.
Anchors: `api/commands/` (pick a destructive command), `ext_commands`.
Plan: follow an existing flag pattern; print counts instead of deleting; unit test both modes.
Interview story: operational safety improvement — seniors love this instinct.

## T16: Regression test for the login enumeration property
Easy · 2-3h
Story: As a maintainer, the "same error for unknown email and wrong password" property is pinned by a test.
Anchors: `account_service.py#L214-L235`.
Plan: two unit tests asserting identical exception type/message; comment linking to the security rationale.
Interview story: encoding a security property as a test — concise and impressive.

---

**Selection advice**: do T1 or T6 first (lowest risk), then one medium (T8/T10/T12). Three shipped tickets across backend, frontend, and tests beats eight backend-only ones — breadth is the mid-level signal.
