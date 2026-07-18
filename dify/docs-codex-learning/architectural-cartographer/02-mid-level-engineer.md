# Mid-Level Engineer Guide

## Architecture Diagram

```text
Browser
  -> Next.js App Router (`web/app`)
    -> global providers (`web/app/layout.tsx:61-84`)
    -> app/workspace context (`web/context/app-context-provider.tsx:30-159`)
    -> components and hooks
      -> React Query / oRPC contracts (`web/service/client.ts:68-137`)
      -> manual fetch wrappers (`web/service/fetch.ts:137-249`, `web/service/base.ts:181-220`)
        -> Flask blueprints (`api/extensions/ext_blueprints.py:25-100`)
          -> controllers (`api/controllers/console/...`)
            -> Pydantic request models
            -> auth/setup/permission decorators
            -> services (`api/services/...`)
              -> SQLAlchemy models (`api/models/...`)
              -> Redis/Celery/storage/provider integrations
```

## Type System Deep Dive

Frontend contracts use TypeScript to describe HTTP boundaries:

```ts
// web/contract/console/apps.ts:6-23
export type AppListQuery = {
  page?: number
  limit?: number
  name?: string
  mode?: AppModeEnum
  tag_ids?: string[]
  is_created_by_me?: boolean
}

export const appListContract = base
  .route({ path: '/apps', method: 'GET' })
  .input(type<{ query?: AppListQuery }>())
  .output(type<AppListResponse>())
```

Backend payloads use Pydantic:

```python
# api/controllers/console/app/app.py:66-93
class AppListQuery(BaseModel):
    page: int = Field(default=1, ge=1, le=99999)
    limit: int = Field(default=20, ge=1, le=100)
    mode: Literal["completion", "chat", "advanced-chat", "workflow", "agent-chat", "channel", "all"] = "all"

    @field_validator("tag_ids", mode="before")
    def validate_tag_ids(cls, value):
        # Converts only valid UUID strings into normalized UUID strings.
        return [str(uuid.UUID(item)) for item in items]
```

Mid-level lesson: frontend types improve caller discipline, but backend validation is still the security and correctness boundary.

## State Management Deep Dive

Workspace state is context-driven:

- Provider loads user, workspace, and system feature data in `web/context/app-context-provider.tsx:30-46`.
- Role booleans are derived in `web/context/app-context-provider.tsx:62-65`.
- Components consume narrow selectors, for example dataset permissions in `web/app/components/datasets/list/index.tsx:28-55`.

Workflow state uses Zustand slices:

```ts
// web/app/components/workflow-app/store/workflow/workflow-slice.ts:3-35
export type WorkflowSliceShape = {
  appId: string
  nodesDefaultConfigs: Record<string, any>
  setNodesDefaultConfigs: (nodesDefaultConfigs: Record<string, any>) => void
}

export const createWorkflowSlice: StateCreator<WorkflowSliceShape> = set => ({
  appId: '',
  nodesDefaultConfigs: {},
  setNodesDefaultConfigs: nodesDefaultConfigs => set(() => ({ nodesDefaultConfigs })),
})
```

The pattern is simple and composable, but `Record<string, any>` is a weak spot because node config shape is not explicit here.

## API Contract Map

- Typed/manual hybrid: `web/contract/router.ts:65-120` aggregates contracts.
- oRPC client: `web/service/client.ts:68-90`.
- Manual request wrapper: `web/service/fetch.ts:181-249`.
- Dataset hooks: `web/service/knowledge/use-dataset.ts:28-104`.
- Workflow hooks: `web/service/use-workflow.ts:21-132`.

## Component Interaction Map

Dataset list:

```text
web/app/(commonLayout)/datasets/page.tsx
  -> web/app/components/datasets/list/index.tsx
    -> local filter state and debounced search
    -> web/app/components/datasets/list/datasets.tsx
      -> useDatasetList()
        -> GET /datasets
```

Workflow app:

```text
WorkflowAppWrapper
  -> WorkflowContextProvider injects createWorkflowSlice
  -> WorkflowAppWithAdditionalContext initializes draft graph
  -> WorkflowAppMain renders canvas/panels
  -> useWorkflowRun sends draft run and consumes streaming events
```

## Full-Stack Feature Trace: App List

1. Frontend component asks for app list through `consoleQuery.apps.list.queryOptions(...)`, as seen in `web/app/account/(commonLayout)/account-page/index.tsx:38-48`.
2. Contract defines query params and response in `web/contract/console/apps.ts:6-23`.
3. `consoleQuery` is produced from `consoleClient` in `web/service/client.ts:87-90`.
4. `consoleClient` sends requests through `OpenAPILink` and the existing `request` wrapper in `web/service/client.ts:68-79`.
5. Backend route is `/apps` in `api/controllers/console/app/app.py:459-460`.
6. Decorators enforce setup, login, account initialization, and enterprise license in `api/controllers/console/app/app.py:465-468`.
7. Pydantic validates query data in `api/controllers/console/app/app.py:473-481`.
8. Service receives typed params in `api/controllers/console/app/app.py:483-485`.
9. `AppService.get_paginate_apps` builds tenant-scoped filters in `api/services/app_service.py:57-94`.
10. SQLAlchemy model shape is `api/models/model.py:392-468`.
11. Controller serializes response via `AppPagination.model_validate(...).model_dump(...)` in `api/controllers/console/app/app.py:531-532`.

## Diff Reading Exercise

Hypothetical change: add an `archived` filter to app list.

Review checklist:

- Frontend contract adds `archived?: boolean` in `web/contract/console/apps.ts:6-13`.
- UI query key includes `archived` if a hook stores app list filters.
- Backend `AppListQuery` validates `archived` in `api/controllers/console/app/app.py:66-74`.
- `AppListParams` gains the field.
- `AppService.get_paginate_apps` applies a filter near `api/services/app_service.py:65-91`.
- Tests cover default behavior and archived filtering.

Red flag: adding the frontend parameter without backend validation silently creates a UI that appears to filter but does not.

## Non-Obvious Patterns

- Bootstrap exemptions: `api/app_factory.py:20-43` exempts selected console endpoints from enterprise license checks to prevent signin/license pages from reload loops.
- Request wrapper reuse: `web/service/client.ts:68-79` lets oRPC use the existing `request` function, preserving auth/redirect behavior.
- Query-cache warmup: `web/context/app-context-provider.tsx:32-39` uses suspense queries so downstream components can read cached data synchronously.
- Hidden model accessors: `api/models/model.py:441-468` makes related data easy to access but can hide database work.

## Mid-Level Socratic Checkpoint

Questions:

1. Why does backend validation still matter if frontend contracts are typed?
2. Where would you add a new app list filter?
3. What query key mistake could make dataset filters stale?
4. Why does `get_app_model` delete `app_id` from kwargs?
5. What is the risk of `App.site` as a property?
6. What happens if workflow draft hash mismatches?
7. Why does the app context provider use suspense?
8. Which layer should know about billing limits?

How to self-grade:

- Strong answers mention backend trust boundaries, cache key completeness, tenant scoping, N+1 query risk, and decorator/service ownership.
- Strong answers cite at least four files: `web/service/knowledge/use-dataset.ts`, `api/controllers/console/app/wraps.py`, `api/services/workflow_service.py:261-318`, and `api/controllers/console/wraps.py:94-133`.

