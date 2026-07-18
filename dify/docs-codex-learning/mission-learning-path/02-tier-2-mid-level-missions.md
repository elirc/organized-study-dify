# Tier 2 Mid-Level Missions

### Mission 9: State Has A Home And A Reason

**Tier:** Mid-Level

**Time Estimate:** 45 minutes

**Goal:** Identify which state belongs in local state, context, React Query, or Zustand.

**The Concept:** State is inventory. In Dify, some inventory belongs on the page, some belongs to the workspace, some belongs to the server cache, and some belongs to the workflow canvas.

**Design Intent Before You Read the Code:** Dataset filters are local UI state. Workspace identity is context. Server data is React Query. Workflow canvas state is Zustand.

**Find It In The Code:** `web/app/components/datasets/list/index.tsx:29-55`, `web/context/app-context-provider.tsx:30-65`, `web/app/components/workflow-app/store/workflow/workflow-slice.ts:3-35`.

```tsx
// web/app/components/datasets/list/index.tsx:35-52
const [keywords, setKeywords] = useState('')
const [searchKeywords, setSearchKeywords] = useState('')
const { run: handleSearch } = useDebounceFn(() => setSearchKeywords(keywords), { wait: 500 })
```

**The Aha Moment:** State placement is an architectural decision, not just a React habit.

**Socratic Checkpoint:** Why are `keywords` and `searchKeywords` separate? Why is workspace role context? Why is dataset list server cache? Why is workflow state not simple local state? What state would be wrong in URL params?

**How to Self-Grade:** Strong answers cite all three files and explain lifetime and sharing scope.

**Connects To:** Mission 10 and Mission 11.

### Mission 10: The Custom Hook Ecosystem

**Tier:** Mid-Level

**Time Estimate:** 50 minutes

**Goal:** Read service hooks as reusable API abstractions.

**The Concept:** Custom hooks are Dify's reusable tools: they package fetching behavior so components do not rebuild HTTP details.

**Design Intent Before You Read the Code:** A hook should expose a stable contract, include complete query keys, gate itself with `enabled` when needed, and centralize invalidation.

**Find It In The Code:** `web/service/knowledge/use-dataset.ts:28-104`, `web/service/use-workflow.ts:21-132`.

```ts
// web/service/knowledge/use-dataset.ts:102-104
export const useInvalidDatasetList = () => {
  return useInvalid([...datasetListQueryKey]) // One invalidation handle for all dataset list variants.
}
```

**The Aha Moment:** Hooks are API affordances; they hide mechanics but should not hide meaning.

**Socratic Checkpoint:** What makes a query key complete? When is `enabled` used? Why centralize invalidation? Which hook supports infinite scrolling? Which hook mutates workflow publish state?

**How to Self-Grade:** Strong answers mention `useInfiniteQuery`, `useMutation`, `useQueryClient`, and `useInvalid`.

**Connects To:** Mission 12.

### Mission 11: Side Effects Are Promises To The System

**Tier:** Mid-Level

**Time Estimate:** 60 minutes

**Goal:** Trace a complex side-effect path.

**The Concept:** Running a workflow is a promise to many systems: UI state, draft sync, SSE, telemetry, abort handling, run history, and backend execution.

**Design Intent Before You Read the Code:** The run hook must clear stale UI state, sync draft changes, validate run mode, prepare callbacks, start a streaming request, and support stopping.

**Find It In The Code:** `web/app/components/workflow-app/hooks/use-workflow-run.ts:191-441`.

```ts
// web/app/components/workflow-app/hooks/use-workflow-run.ts:202-209
const newNodes = produce(getNodes(), (draft: Node[]) => {
  draft.forEach((node) => {
    node.data.selected = false
    node.data._runningStatus = undefined
  })
})
setNodes(newNodes)
await doSyncWorkflowDraft()
```

**The Aha Moment:** Side effects must be ordered because each one changes what the next one can safely assume.

**Socratic Checkpoint:** Why clear node status before running? Why sync draft before SSE? Where is abort state stored? Where is telemetry tracked? What breaks if stop does not clear listening state?

**How to Self-Grade:** Strong answers cite `web/app/components/workflow-app/hooks/use-workflow-run.ts:256-441`.

**Connects To:** Mission 14.

### Mission 12: The Full API Contract

**Tier:** Mid-Level

**Time Estimate:** 55 minutes

**Goal:** Map frontend contract to backend validation.

**The Concept:** API contracts are handshakes. Both sides must agree on method, path, input, output, validation, and errors.

**Design Intent Before You Read the Code:** The frontend app contract declares `/apps`; the backend route validates the same query and returns a Pydantic response model.

**Find It In The Code:** `web/contract/console/apps.ts:6-35`, `api/controllers/console/app/app.py:66-93`, `api/controllers/console/app/app.py:459-561`.

```ts
// web/contract/console/apps.ts:15-23
base.route({ path: '/apps', method: 'GET' })
  .input(type<{ query?: AppListQuery }>())
  .output(type<AppListResponse>())
```

```python
# api/controllers/console/app/app.py:473-485
args = AppListQuery.model_validate(_normalize_app_list_query_args(request.args))
params = AppListParams(...)
app_pagination = AppService().get_paginate_apps(current_user.id, current_tenant_id, params)
```

**The Aha Moment:** A full API contract includes runtime validation, not just static types.

**Socratic Checkpoint:** What is the HTTP method? What validates query params? What serializes output? Which layer knows SQL? Which error is likely for invalid tag UUID?

**How to Self-Grade:** Strong answers explicitly separate TypeScript contract, Pydantic validation, service filtering, and response serialization.

**Connects To:** Mission 13.

### Mission 13: The Middleware Or Boundary Chain

**Tier:** Mid-Level

**Time Estimate:** 45 minutes

**Goal:** Explain decorators and request hooks as boundary protection.

**The Concept:** Boundaries are like workflow guard nodes: they stop invalid execution before business logic runs.

**Design Intent Before You Read the Code:** Request hooks add enterprise license/tracing behavior. Decorators enforce setup, login, account initialization, billing, app ownership, and permissions.

**Find It In The Code:** `api/app_factory.py:58-118`, `api/libs/login.py:85-100`, `api/controllers/console/app/wraps.py:13-84`.

```python
# api/libs/login.py:90-100
if user is None or not user.is_authenticated:
    return _get_login_manager().unauthorized()
g._login_user = user
check_csrf_token(request, user.id)
return current_app.ensure_sync(func)(*args, **kwargs)
```

**The Aha Moment:** Security and setup checks are most maintainable when they are reusable boundary code.

**Socratic Checkpoint:** Where is CSRF checked? Where is tenant-scoped app loading? What endpoints are enterprise-exempt? Why do decorators keep controllers readable? What should a new endpoint reuse?

**How to Self-Grade:** Strong answers cite all three files and mention tenant isolation.

**Connects To:** Mission 22.

### Mission 14: End-To-End Feature Trace

**Tier:** Mid-Level

**Time Estimate:** 75 minutes

**Goal:** Trace app list from UI/client to database and back.

**The Concept:** End-to-end tracing is debugging before the bug happens.

**Design Intent Before You Read the Code:** Each layer should transform or protect data once, then pass it along.

**Find It In The Code:** `web/contract/console/apps.ts:6-23`, `web/service/client.ts:68-137`, `api/controllers/console/app/app.py:459-532`, `api/services/app_service.py:56-100`, `api/models/model.py:392-468`.

```python
# api/services/app_service.py:65-98
filters = [App.tenant_id == tenant_id, App.is_universal == False]
if params.name:
    escaped_name = escape_like_pattern(params.name[:30])
    filters.append(App.name.ilike(f"%{escaped_name}%", escape="\\"))
return db.paginate(sa.select(App).where(*filters).order_by(App.created_at.desc()), ...)
```

**The Aha Moment:** The tenant filter is the security spine of the app list flow.

**Socratic Checkpoint:** Where does path `/apps` come from? Where are query params validated? Where is SQL built? Where is tenant enforced? Where does the response become JSON?

**How to Self-Grade:** Strong answers explain all five layers in order without skipping service logic.

**Connects To:** Mission 15.

### Mission 15: Read The Diff Like An Engineer

**Tier:** Mid-Level

**Time Estimate:** 45 minutes

**Goal:** Practice reviewing a hypothetical change.

**The Concept:** A diff is a proposed story about the system. Your job is to find where the story is incomplete.

**Design Intent Before You Read the Code:** Adding `archived` to app list should change contract, backend validation, service filters, tests, and cache keys.

**Find It In The Code:** Study `web/contract/console/apps.ts:6-23`, `api/controllers/console/app/app.py:66-93`, and `api/services/app_service.py:56-100`.

```text
Hypothetical diff:
- Add `archived?: boolean` to AppListQuery.
- Add `archived` to backend AppListQuery.
- Add filter in AppService.
- Add tests for default, true, false.
```

**The Aha Moment:** Most bugs in feature diffs are missing-layer bugs.

**Socratic Checkpoint:** Which layer is easiest to forget? What test proves backward compatibility? What should happen if backend gets unknown params? What cache key might need updating? What permission rule must remain unchanged?

**How to Self-Grade:** Strong answers describe a complete diff and at least three failure modes.

**Connects To:** Mission 19.

### Mission 16: Composition Over Inheritance

**Tier:** Mid-Level

**Time Estimate:** 40 minutes

**Goal:** Identify composition patterns in frontend and backend.

**The Concept:** Composition lets Dify assemble apps, providers, and workflows from smaller parts instead of one giant class.

**Design Intent Before You Read the Code:** Frontend providers compose capabilities. Backend extension loading composes infrastructure.

**Find It In The Code:** `web/app/layout.tsx:61-84`, `api/app_factory.py:177-219`, `web/app/components/workflow-app/index.tsx:142-154`.

```python
# api/app_factory.py:177-207
extensions = [ext_timezone, ext_logging, ext_database, ext_redis, ext_storage, ext_celery, ext_login, ext_blueprints]
for ext in extensions:
    ext.init_app(app) # each extension owns one setup concern
```

**The Aha Moment:** Composition makes infrastructure order visible and testable.

**Socratic Checkpoint:** Which providers wrap the frontend? Why is extension order explicit? What is injected into `WorkflowContextProvider`? Why is that better than hardcoding one store? What risk remains?

**How to Self-Grade:** Strong answers cite provider and extension examples.

**Connects To:** Mission 18.

### Mission 17: TypeScript's Hidden Work

**Tier:** Mid-Level

**Time Estimate:** 45 minutes

**Goal:** Find where TypeScript protects you and where it cannot.

**The Concept:** TypeScript is a bright flashlight, but runtime data still comes from the dark.

**Design Intent Before You Read the Code:** Contract types, context types, and hook response types improve safety. `any` weakens boundaries and should be replaced only with real domain shapes.

**Find It In The Code:** `web/service/fetch.ts:21-24`, `web/context/app-context.ts:7-73`, `web/app/components/workflow-app/store/workflow/workflow-slice.ts:3-18`.

```ts
// web/service/fetch.ts:21-24
export type FetchOptionType = Omit<RequestInit, 'body'> & {
  params?: Record<string, any> // flexible but weak
  body?: BodyInit | Record<string, any> | null
}
```

**The Aha Moment:** Strong TypeScript is strongest at boundaries, but only if the boundary types are precise.

**Socratic Checkpoint:** Where is `any` useful? Where is it risky? Which context values are strongly typed? Which workflow config shape could be narrowed? Why not blindly replace all `any` with complex types?

**How to Self-Grade:** Strong answers are nuanced: they identify risk without proposing churn.

**Connects To:** Mission 23.

