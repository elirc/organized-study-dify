# User Stories

## Story 1: Harden API Method Tag Styling

**Difficulty:** Easy

**Estimated Time:** 1 hour

**Skills You'll Practice:** React props, TypeScript literals, unit testing, defensive UI

**The Story:** As a developer reading API docs, I want unknown method tag colors to be handled safely so that documentation UI never crashes from unexpected input.

**Why This Story Matters:** `Tag` is small, but it demonstrates how visual components encode product conventions and how weak prop types can hide runtime risk.

**Acceptance Criteria:**
- [ ] `Tag` accepts only supported color and variant values at the type level.
- [ ] Unknown method text still falls back to the default color.
- [ ] Passing an unsupported color cannot crash at runtime.
- [ ] Existing tests still pass and at least one defensive test is added.

**Files You'll Likely Touch:**
- `web/app/components/develop/tag.tsx` - component and prop types.
- `web/app/components/develop/__tests__/tag.spec.tsx` - behavior tests.

**Relevant Existing Patterns:**
- `web/app/components/develop/tag.tsx:4-60`
- `web/app/components/develop/__tests__/tag.spec.tsx:18-116`

**High-Level Implementation Plan:**
1. In `tag.tsx`, replace loose `string` prop fields with literal union types derived from style maps.
2. Add a safe lookup path for `colorStyles[color]?.[variant]`.
3. Preserve fallback color behavior for unknown `children` values.
4. Add tests for unknown method and explicit unsupported value if you expose a fallback path.

**Testing and Verification Plan:**
Run `pnpm -C web test web/app/components/develop/__tests__/tag.spec.tsx`.

**Tips:**
- Keep the component simple; do not introduce a new styling abstraction.
- Use the current color map in `tag.tsx:36-41`.
- Tests should assert user-observable class behavior, as current tests do.
- Avoid changing visual classes unless the story requires it.

**What Could Go Wrong:**
- Overly strict types make existing callers fail.
- Runtime fallback is removed accidentally.
- Tests assert exact full class strings and become brittle.

**Stretch Goal:** Add table-driven tests for all method-to-color mappings.

**Connects To:** Story 2, because both improve small UI correctness.

## Story 2: Add A Dataset Empty Search State

**Difficulty:** Easy

**Estimated Time:** 2 hours

**Skills You'll Practice:** React conditional rendering, i18n, React Query data shape

**The Story:** As a workspace user, I want a clear empty state when dataset filters return no results so that I know whether to change filters or create knowledge.

**Why This Story Matters:** Empty states are product guidance. Dify's dataset list already has filters and infinite loading, so a no-results state reduces confusion.

**Acceptance Criteria:**
- [ ] When dataset pages contain no datasets and fetching is complete, an empty state appears.
- [ ] Empty state copy uses `web/i18n/en-US/`.
- [ ] The new state does not hide `NewDatasetCard` for editors.
- [ ] Infinite scrolling still works when results exist.

**Files You'll Likely Touch:**
- `web/app/components/datasets/list/datasets.tsx` - render empty state.
- `web/i18n/en-US/dataset.ts` or related dataset locale file - copy.

**Relevant Existing Patterns:**
- `web/app/components/datasets/list/datasets.tsx:26-68`
- `web/app/components/datasets/list/index.tsx:35-52`
- `web/AGENTS.md` user-facing string rule.

**High-Level Implementation Plan:**
1. Compute whether all pages contain zero dataset items in `datasets.tsx`.
2. Render an existing empty-state/base component if one exists in nearby dataset components.
3. Keep `NewDatasetCard` visible when the current user can create datasets.
4. Add i18n keys for title/body.

**Testing and Verification Plan:**
Add/update a component test for empty results if a sibling dataset list test exists. Manually verify with a search keyword that returns no results.

**Tips:**
- `datasetList?.pages.map` is the current render path in `datasets.tsx:64-66`.
- Keep `isFetchingNextPage` loading behavior unchanged.
- Do not hardcode English strings in JSX.

**What Could Go Wrong:**
- Empty state appears during first loading flicker.
- Empty state hides the create card.
- i18n key is added in the wrong namespace.

**Stretch Goal:** Show a different empty message when filters are active versus no datasets exist.

**Connects To:** Story 3, because it also refines dataset list behavior.

## Story 3: Add Dataset Search Minimum Length Feedback

**Difficulty:** Easy

**Estimated Time:** 2 hours

**Skills You'll Practice:** form UX, debounce behavior, i18n, component tests

**The Story:** As a dataset user, I want feedback when my search term is too short so that I understand why results are not changing.

**Why This Story Matters:** The dataset list debounces search input in `web/app/components/datasets/list/index.tsx:35-43`. Adding small validation teaches the difference between display input and effective query input.

**Acceptance Criteria:**
- [ ] Search terms shorter than a chosen threshold do not update `searchKeywords`.
- [ ] A localized helper message appears only when the threshold blocks search.
- [ ] Clearing the input resets the search.
- [ ] Existing tag and include-all filters continue working.

**Files You'll Likely Touch:**
- `web/app/components/datasets/list/index.tsx` - search behavior and helper display.
- `web/i18n/en-US/dataset.ts` - helper text.

**Relevant Existing Patterns:**
- `web/app/components/datasets/list/index.tsx:35-52`
- `web/service/knowledge/use-dataset.ts:83-99`

**High-Level Implementation Plan:**
1. Choose a threshold consistent with product expectations.
2. In `handleSearch`, update `searchKeywords` only for empty input or valid length.
3. Add local state or derived boolean for helper visibility.
4. Pass unchanged `searchKeywords` to `<Datasets />`.
5. Add tests for too-short, valid, and cleared input.

**Testing and Verification Plan:**
Use Vitest/RTL if a nearby test exists. Otherwise manually test debounce with short and valid terms.

**Tips:**
- Preserve the 500ms debounce in `index.tsx:37-39`.
- Do not put validation in `useDatasetList`; the hook should receive intended query params.
- Use user-facing strings from i18n.

**What Could Go Wrong:**
- Helper appears before typing.
- Clearing input no longer fetches all datasets.
- Debounced state closes over stale `keywords`.

**Stretch Goal:** Add an accessible `aria-live` region for the helper text.

**Connects To:** Story 4, because it builds list-filter confidence.

## Story 4: Show Dataset API Availability On Cards

**Difficulty:** Medium

**Estimated Time:** 3 hours

**Skills You'll Practice:** reading response models, card UI, conditional rendering, tests

**The Story:** As a workspace manager, I want dataset cards to show whether service API access is enabled so that I can spot API-ready knowledge bases quickly.

**Why This Story Matters:** Dataset API capability exists in backend data (`enable_api` on `Dataset`) and service API panels. Surfacing it in the list improves operational visibility.

**Acceptance Criteria:**
- [ ] Dataset cards show an API-enabled/API-disabled indicator.
- [ ] Indicator is visible only when the response contains the field.
- [ ] Styling follows existing card/tag patterns.
- [ ] No extra network request is added per card.

**Files You'll Likely Touch:**
- `web/app/components/datasets/list/dataset-card.tsx` - display indicator.
- `web/models/datasets.ts` - ensure type includes API flag if missing.
- `web/i18n/en-US/dataset.ts` - label text.

**Relevant Existing Patterns:**
- `api/models/dataset.py:166-212`
- `api/controllers/console/datasets/datasets.py:342-379`
- `web/app/components/datasets/list/datasets.tsx:64-66`

**High-Level Implementation Plan:**
1. Inspect dataset card props and dataset type.
2. Confirm `enable_api` is present in the response field mapping.
3. Add a compact indicator using existing badge/tag styles.
4. Add localized tooltip or text.
5. Add a card test if a dataset card test exists.

**Testing and Verification Plan:**
Mock a dataset with `enable_api: true` and `false`; assert the indicator text/label.

**Tips:**
- Avoid calling `useDatasetApiBaseUrl` per card.
- Follow existing list card spacing.
- Keep the indicator informational, not an action.

**What Could Go Wrong:**
- Type says field exists but backend field mapping omits it.
- UI creates visual noise in dense grids.
- Tests overmock child components.

**Stretch Goal:** Add a filter for API-enabled datasets after the indicator ships.

**Connects To:** Story 6, because API availability can become a query/API feature.

## Story 5: Add App List Created-By-Me Filter To A UI Surface

**Difficulty:** Medium

**Estimated Time:** 3 hours

**Skills You'll Practice:** typed query params, UI controls, React Query

**The Story:** As an app builder, I want to filter app lists to apps I created so that I can focus on my own work.

**Why This Story Matters:** The backend and contract already support `is_created_by_me`, but a UI surface should expose it consistently.

**Acceptance Criteria:**
- [ ] A UI control toggles created-by-me filtering.
- [ ] The request includes `is_created_by_me`.
- [ ] Query cache differentiates filtered and unfiltered results.
- [ ] State resets predictably when leaving the page.

**Files You'll Likely Touch:**
- The app list UI component you identify under `web/app/components/app` or route page.
- `web/contract/console/apps.ts` only if the type is incomplete.
- Tests near the app list component.

**Relevant Existing Patterns:**
- `web/contract/console/apps.ts:6-23`
- `api/controllers/console/app/app.py:473-485`
- `api/services/app_service.py:78-79`

**High-Level Implementation Plan:**
1. Locate the app list component that calls `consoleQuery.apps.list`.
2. Add a checkbox/toggle following existing UI patterns.
3. Include `is_created_by_me` in the query input.
4. Ensure query key includes the input through `consoleQuery` utilities.
5. Add a test that toggling changes the service query input.

**Testing and Verification Plan:**
Mock app list query and assert params include `is_created_by_me: true`.

**Tips:**
- Backend already filters `App.created_by == user_id` in `api/services/app_service.py:78-79`.
- Use i18n for new copy.
- Keep filter local unless URL persistence already exists in that list.

**What Could Go Wrong:**
- UI state changes but query input does not.
- Query key does not differentiate results.
- Filter appears for a list that is not user-specific.

**Stretch Goal:** Persist the filter in URL query params using existing `nuqs` patterns.

**Connects To:** Story 8, because it prepares you to add backend-supported filters.

## Story 6: Add Dataset API-Enabled Filter

**Difficulty:** Medium

**Estimated Time:** 5 hours

**Skills You'll Practice:** API query design, React Query keys, Pydantic validation, SQLAlchemy filters

**The Story:** As a workspace manager, I want to filter datasets by service API enabled status so that I can manage externally accessible knowledge bases.

**Why This Story Matters:** This crosses UI, hook query params, backend validation, service filtering, and tests.

**Acceptance Criteria:**
- [ ] Dataset list supports `api_enabled=true/false` filter.
- [ ] Frontend query key includes the new filter.
- [ ] Backend validates the query param.
- [ ] SQL query filters by `Dataset.enable_api`.
- [ ] Tests cover true, false, and absent behavior.

**Files You'll Likely Touch:**
- `web/app/components/datasets/list/index.tsx` - filter control.
- `web/app/components/datasets/list/datasets.tsx` - prop passthrough.
- `web/service/knowledge/use-dataset.ts` - params/query key.
- `web/models/datasets.ts` - request type.
- `api/controllers/console/datasets/datasets.py` - Pydantic query model.
- `api/services/dataset_service.py` - dataset filtering.

**Relevant Existing Patterns:**
- `web/service/knowledge/use-dataset.ts:83-99`
- `api/controllers/console/datasets/datasets.py:199-207`
- `api/models/dataset.py:210-212`

**High-Level Implementation Plan:**
1. Add optional API-enabled state in dataset list UI.
2. Pass it to `Datasets` and `useDatasetList`.
3. Include it in query key and `qs.stringify`.
4. Add the query field to `ConsoleDatasetListQuery`.
5. Find `DatasetService.get_datasets` and add a conditional filter.
6. Add frontend and backend tests.

**Testing and Verification Plan:**
Backend unit test or focused controller/service test for filter behavior. Frontend test for query params.

**Tips:**
- Avoid string booleans leaking past validation.
- Preserve existing `include_all`, tags, and keyword behavior.
- Keep the backend field optional so old callers are unaffected.

**What Could Go Wrong:**
- Query param arrives as string and is treated truthy for both `true` and `false`.
- Query key omits the filter.
- Dataset permissions are bypassed by a new service branch.

**Stretch Goal:** Combine with Story 4 indicator for a complete management workflow.

**Connects To:** Story 8, because this is a smaller full-stack filter.

## Story 7: Add Workflow Run Validation Feedback Before SSE

**Difficulty:** Medium

**Estimated Time:** 5 hours

**Skills You'll Practice:** complex hooks, validation UX, workflow debugging, tests

**The Story:** As a workflow builder, I want clear feedback when a run cannot start so that I can fix inputs before waiting on streaming output.

**Why This Story Matters:** `useWorkflowRun` already validates run requests and logs errors. User-facing feedback would improve the debugging loop.

**Acceptance Criteria:**
- [ ] Validation failures produce localized UI feedback instead of console-only errors.
- [ ] SSE is not started when validation fails.
- [ ] Existing trigger debug modes still work.
- [ ] Tests cover validation failure and successful run.

**Files You'll Likely Touch:**
- `web/app/components/workflow-app/hooks/use-workflow-run.ts` - validation branch.
- `web/app/components/workflow-app/hooks/use-workflow-run-utils.ts` - validation messages if needed.
- `web/i18n/en-US/workflow.ts` or related namespace - localized feedback.
- `web/app/components/workflow-app/hooks/__tests__/use-workflow-run.spec.ts` - tests.

**Relevant Existing Patterns:**
- `web/app/components/workflow-app/hooks/use-workflow-run.ts:244-254`
- `web/app/components/workflow-app/hooks/use-workflow-run.ts:435-441`
- `web/app/components/workflow-app/hooks/__tests__/use-workflow-run.spec.ts:32-180`

**High-Level Implementation Plan:**
1. Inspect `validateWorkflowRunRequest` output shape.
2. Replace or supplement `console.error(validationMessage)` with an existing toast/notification pattern.
3. Ensure early return still prevents `ssePost`.
4. Add tests asserting validation feedback and `mockSsePost` not called.

**Testing and Verification Plan:**
Run the workflow run hook test. Manually trigger a validation failure in workflow debug if feasible.

**Tips:**
- Do not alter the successful run path.
- Keep validation utility pure if it is currently pure.
- Use existing toast/import patterns.

**What Could Go Wrong:**
- Toast fires repeatedly on rerender.
- A validation branch accidentally starts SSE.
- Tests become too coupled to implementation internals.

**Stretch Goal:** Add field-level focus/scroll to the missing input.

**Connects To:** Story 10, because workflow run observability is an architectural theme.

## Story 8: Add App Archived Filter End-To-End

**Difficulty:** Hard

**Estimated Time:** 8 hours

**Skills You'll Practice:** full-stack filtering, database model reading, API validation, tests

**The Story:** As a workspace admin, I want to filter archived apps separately from active apps so that workspace cleanup is safer.

**Why This Story Matters:** App lifecycle state already exists through `App.status`; exposing it carefully trains full-stack feature work without inventing a new domain object.

**Acceptance Criteria:**
- [ ] Frontend app list can request active, archived, or all apps.
- [ ] Backend validates the requested status filter.
- [ ] Service applies the filter without breaking default behavior.
- [ ] Tenant scoping remains intact.
- [ ] Tests cover default and archived cases.

**Files You'll Likely Touch:**
- `web/contract/console/apps.ts` - app list query type.
- App list UI component - filter control.
- `api/controllers/console/app/app.py` - `AppListQuery` and params.
- `api/services/app_service.py` - status filter.
- Backend/frontend tests.

**Relevant Existing Patterns:**
- `api/models/model.py:406-408`
- `api/services/app_service.py:65-100`
- `api/controllers/console/app/app.py:66-93`
- `web/contract/console/apps.ts:6-23`

**High-Level Implementation Plan:**
1. Decide product vocabulary: active/archived/all.
2. Add frontend query type field.
3. Add UI control with i18n.
4. Add backend Pydantic literal field.
5. Add service filter while preserving `App.tenant_id == tenant_id`.
6. Add tests for default behavior and archived behavior.

**Testing and Verification Plan:**
Backend service/controller tests plus UI query-param test.

**Tips:**
- Check current meaning of `App.status`.
- Default should match existing behavior.
- Avoid changing delete/archive semantics unless required.

**What Could Go Wrong:**
- Archived apps leak across tenants.
- Default behavior changes and users see archived apps unexpectedly.
- UI filter names do not match backend enum values.

**Stretch Goal:** Add an archived count next to the filter.

**Connects To:** Story 9, because both touch app lifecycle.

## Story 9: Add App List Last Updated Metadata

**Difficulty:** Hard

**Estimated Time:** 8 hours

**Skills You'll Practice:** response DTOs, SQLAlchemy fields, frontend rendering, date formatting

**The Story:** As a workspace user, I want app lists to show when each app was last updated so that I can identify stale applications.

**Why This Story Matters:** Metadata visibility helps users operate real workspaces and teaches response-shape changes.

**Acceptance Criteria:**
- [ ] App list response includes `updated_at` if not already exposed.
- [ ] Frontend displays formatted last-updated text.
- [ ] Sorting remains unchanged unless explicitly requested.
- [ ] Tests cover rendering and response shape.

**Files You'll Likely Touch:**
- `api/controllers/console/app/app.py` - response model if missing field.
- `api/models/model.py` - source field already exists.
- Frontend app list card/table component.
- `web/models/app.ts` - app response type if needed.
- i18n file for label text.

**Relevant Existing Patterns:**
- `api/models/model.py:419-423`
- `api/controllers/console/app/app.py:323-403`
- `api/controllers/console/app/app.py:531-532`

**High-Level Implementation Plan:**
1. Inspect `AppPartial` and `AppPagination` response models.
2. Add/confirm `updated_at` serialization.
3. Update frontend type.
4. Render formatted date in the app list card/table.
5. Add tests for present and missing updated date.

**Testing and Verification Plan:**
Backend serialization test and frontend rendering test.

**Tips:**
- Use existing date formatting utilities.
- Avoid adding expensive related queries.
- Keep timezone behavior consistent with user profile if existing helpers support it.

**What Could Go Wrong:**
- Backend returns datetime in a format frontend does not expect.
- Timezone display is inconsistent.
- Response model change breaks a typed contract.

**Stretch Goal:** Add optional sort by recently updated.

**Connects To:** Story 10, because operational metadata often leads to audit logging.

## Story 10: Add Workflow Run Audit Events

**Difficulty:** Expert

**Estimated Time:** 16 hours

**Skills You'll Practice:** architecture, event modeling, background execution, persistence, security, tests

**The Story:** As a workspace admin, I want workflow debug/run actions to emit audit events so that important execution activity can be reviewed later.

**Why This Story Matters:** Workflow execution is central and sensitive. Auditability crosses frontend triggers, backend routes, user identity, app/workflow ids, and persistence or event dispatch.

**Acceptance Criteria:**
- [ ] Starting a workflow debug run records who started it, app id, workflow id or draft context, timestamp, and run mode.
- [ ] Stopping a workflow run records who stopped it and task id.
- [ ] Audit writes are tenant-scoped.
- [ ] Audit failures do not break workflow execution unless explicitly configured.
- [ ] Tests cover start, stop, and failure isolation.

**Files You'll Likely Touch:**
- `api/controllers/console/app/workflow.py` - start/stop route boundaries.
- `api/services/workflow_service.py` or a new narrowly scoped audit service - business logic.
- `api/models/...` - audit persistence if no existing model exists.
- `api/migrations/...` - database migration if persistence is added.
- `web/app/components/workflow-app/hooks/use-workflow-run.ts` - only if frontend run mode metadata must be explicit.
- Backend tests under `api/tests/`.

**Relevant Existing Patterns:**
- `api/controllers/console/app/workflow.py:697-759`
- `web/app/components/workflow-app/hooks/use-workflow-run.ts:191-441`
- `api/services/workflow_service.py:261-318`
- `api/models/base.py:26-91`
- `api/tests/unit_tests/tasks/test_workflow_execute_task.py:73-160`

**High-Level Implementation Plan:**
1. Search for an existing audit/event/log model or service before adding a new one.
2. Decide whether audit events belong in DB, event bus, or existing logstore.
3. Define a narrow service API such as `record_workflow_run_started(...)`.
4. Call it from workflow start and stop routes after auth/tenant decorators have resolved user/app.
5. If adding persistence, create a migration and SQLAlchemy model using existing base patterns.
6. Make audit failures logged and non-blocking unless product requirements say otherwise.
7. Add tests for start/stop and error isolation.

**Testing and Verification Plan:**
Backend unit tests for service behavior and controller integration. Manual verification by running/stopping a workflow and checking audit storage.

**Tips:**
- Do not put persistence details directly in the controller.
- Preserve streaming response behavior.
- Keep tenant id in every audit row.
- Treat user id, app id, task id, and run mode as required fields.
- Review existing logging/event services before inventing storage.

**What Could Go Wrong:**
- Audit write slows or breaks workflow execution.
- Stop events are missing when task id is absent and abort is frontend-only.
- Tenant id is omitted, making audit data unsafe to query.

**Stretch Goal:** Add a console UI tab for recent workflow audit events.

**Connects To:** This is the capstone story; it combines routing, auth, workflow execution, persistence, and tests.

