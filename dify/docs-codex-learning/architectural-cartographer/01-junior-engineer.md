# Junior Engineer Guide

## Setup And Run

Found setup sources:

- Root package manager: `package.json:1-18` uses `pnpm@11.1.1` and Node `^22.22.1`.
- Frontend scripts: `web/package.json:25-43`.
- Backend setup: `Makefile:17-46`.
- Backend quality/test commands: `Makefile:66-105`.
- Frontend test guide: `web/docs/test.md:7-23`.
- Frontend lint/type guide: `web/docs/lint.md:48-77`.

Likely local setup:

1. Install Node 22.22.x and pnpm 11.1.1.
2. Install `uv` for Python 3.12.
3. Copy env files:
   - `api/.env.example` to `api/.env`
   - `web/.env.example` to `web/.env.local`
   - `docker/envs/middleware.env.example` to `docker/middleware.env`
4. Start middleware with `cd docker && docker compose -f docker-compose.middleware.yaml --env-file middleware.env -p dify-middlewares-dev up -d`.
5. Install frontend dependencies with `pnpm install`.
6. Sync backend dependencies with `uv sync --project api --dev`.
7. Run backend migrations with `uv run --project api flask db upgrade`.
8. Start frontend with root `pnpm dev` or `pnpm -C web dev:vinext` plus proxy.
9. Backend long-running service start exists through `api/app.py`, but repo agent instructions say not to start long-running backend services during agent work.

Assumptions: this reconstructs the path from `Makefile:17-46`; exact production secrets, model providers, and external integrations require local env values.

Common commands:

- Frontend tests: `pnpm -C web test` from `web/docs/test.md:7-23`.
- Frontend type check: `pnpm -C web type-check` from `web/docs/lint.md:67-77`.
- Backend unit tests: `make test TARGET_TESTS=./api/tests/<target_tests>` from `Makefile:97-105`.
- Backend lint/type: `make lint`, `make type-check` from `Makefile:77-89`.

## Folder Orientation

Root:

- `api/`: Flask backend, controllers, services, models, tasks, repositories, providers.
- `web/`: Next.js frontend, React components, app routes, hooks, service clients, i18n.
- `e2e/`: Cucumber and Playwright tests (`e2e/package.json:5-15`).
- `docker/`: compose files and middleware/services.
- `packages/`: shared TypeScript packages, contracts, UI packages.
- `dify-agent/`: agent backend package.

Frontend key folders:

- `web/app/`: Next App Router pages, route groups, and most UI surfaces.
- `web/service/`: HTTP wrappers, React Query hooks, typed clients.
- `web/context/`: providers for app, workspace, modal, query client, debug, events.

Backend key folders:

- `api/controllers/`: Flask-RESTX resources split by audience.
- `api/services/`: business logic and side effects.
- `api/models/`: SQLAlchemy persistence models and typed dict payload shapes.

## Entry Point Walkthroughs

Backend entry:

```python
# api/app.py:32-62
flask_app = None
socketio_app = None

if is_db_command():
    # Migration commands use a smaller app: database + migrations only.
    from app_factory import create_migrations_app
    app = create_migrations_app()
else:
    # Normal runtime builds Flask plus Socket.IO and Celery extension access.
    from app_factory import create_app
    socketio_app, flask_app = create_app()
    app = flask_app
    celery = cast("Celery", app.extensions["celery"])

if __name__ == "__main__":
    # Local script mode serves through gevent WebSocket support.
    server = pywsgi.WSGIServer((HOST, PORT), socketio_app, handler_class=WebSocketHandler)
    server.serve_forever()
```

Frontend entry:

```tsx
// web/app/layout.tsx:61-84
<JotaiProvider>
  <ThemeProvider attribute="data-theme" defaultTheme="system">
    <NuqsAdapter>
      <TanstackQueryInitializer>
        <I18nServerProvider>
          <ToastHost timeout={5000} limit={3} />
          <TooltipProvider delay={300} closeDelay={200}>
            {children}
          </TooltipProvider>
        </I18nServerProvider>
      </TanstackQueryInitializer>
    </NuqsAdapter>
  </ThemeProvider>
</JotaiProvider>
```

The frontend root is less about rendering content and more about giving every page access to state, theme, query cache, i18n, toasts, and tooltips.

## TypeScript Orientation

This repo uses TypeScript as a boundary language:

- API contract type: `web/contract/console/apps.ts:6-23`.
- Query response type: `web/service/knowledge/use-dataset.ts:66-80`.
- Context value type: `web/context/app-context.ts:7-73`.
- Workflow slice type: `web/app/components/workflow-app/store/workflow/workflow-slice.ts:3-18`.

Beginner pattern to notice: types often describe "what this layer promises" rather than only "what this variable is."

## Three React Component Anatomies

Simple component:

```tsx
// web/app/components/develop/tag.tsx:43-60
type ITagProps = {
  children: string
  color?: string
  variant?: string
}

export function Tag({ children, variant = 'medium', color = valueColorMap[children.toLowerCase()] ?? 'emerald' }: ITagProps) {
  return (
    <span className={cn('font-mono text-[0.625rem] leading-6 font-semibold', variantStyles[variant], colorStyles[color]![variant])}>
      {children}
    </span>
  )
}
```

What to learn: props become display contracts. A risky detail is `colorStyles[color]!`; an invalid color can crash at runtime even though TypeScript allows `string`.

Data-list component:

```tsx
// web/app/components/datasets/list/datasets.tsx:26-68
const { data: datasetList, fetchNextPage, hasNextPage, isFetching, isFetchingNextPage } = useDatasetList({
  initialPage: 1,
  tag_ids: tags,
  limit: 30,
  include_all: includeAll,
  keyword: keywords,
})

useEffect(() => {
  // IntersectionObserver turns scrolling near the anchor into fetchNextPage().
  if (anchorRef.current) {
    observerRef.current = new IntersectionObserver((entries) => {
      if (entries[0]!.isIntersecting && hasNextPage && !isFetching)
        fetchNextPage()
    }, { rootMargin: '100px' })
    observerRef.current.observe(anchorRef.current)
  }
  return () => observerRef.current?.disconnect()
}, [anchorRef, hasNextPage, isFetching, fetchNextPage])
```

What to learn: UI state (`tags`, `keywords`) shapes the query key and backend request.

Complex component:

```tsx
// web/app/components/workflow-app/index.tsx:40-155
const { data, isLoading, fileUploadConfigResponse } = useWorkflowInit()
const nodesData = useMemo(() => data ? initialNodes(data.graph.nodes, data.graph.edges) : [], [data])
const edgesData = useMemo(() => data ? initialEdges(data.graph.edges, data.graph.nodes) : [], [data])

if (!data || isLoading || isLoadingCurrentWorkspace || !currentWorkspace.id)
  return <Loading />

return (
  <WorkflowWithDefaultContext edges={edgesData} nodes={nodesData}>
    <FeaturesProvider features={buildInitialFeatures(data.features, fileUploadConfigResponse)}>
      <WorkflowAppMain nodes={nodesData} edges={edgesData} viewport={data.graph.viewport} />
    </FeaturesProvider>
  </WorkflowWithDefaultContext>
)
```

What to learn: complex components often coordinate data, stores, providers, and feature flags before rendering a specialized child.

## Three Backend Route Anatomies

App list:

```python
# api/controllers/console/app/app.py:469-532
current_user, current_tenant_id = current_account_with_tenant()
args = AppListQuery.model_validate(_normalize_app_list_query_args(request.args))
params = AppListParams(page=args.page, limit=args.limit, mode=args.mode, name=args.name, tag_ids=args.tag_ids)
app_pagination = AppService().get_paginate_apps(current_user.id, current_tenant_id, params)
pagination_model = AppPagination.model_validate(app_pagination, from_attributes=True)
return pagination_model.model_dump(mode="json"), 200
```

Dataset list:

```python
# api/controllers/console/datasets/datasets.py:289-320
@console_ns.route("/datasets")
class DatasetListApi(Resource):
    @setup_required
    @login_required
    @account_initialization_required
    @enterprise_license_required
    def get(self):
        current_user, current_tenant_id = current_account_with_tenant()
        query_params = dict(request.args.to_dict())
        query = ConsoleDatasetListQuery.model_validate(query_params)
```

Workflow draft run:

```python
# api/controllers/console/app/workflow.py:697-733
@console_ns.route("/apps/<uuid:app_id>/workflows/draft/run")
class DraftWorkflowRunApi(Resource):
    @get_app_model(mode=[AppMode.WORKFLOW])
    @edit_permission_required
    def post(self, app_model: App):
        current_user, _ = current_account_with_tenant()
        args = DraftWorkflowRunPayload.model_validate(console_ns.payload or {}).model_dump(exclude_none=True)
        response = AppGenerateService.generate(app_model=app_model, user=current_user, args=args, invoke_from=InvokeFrom.DEBUGGER, streaming=True)
        return helper.compact_generate_response(response)
```

## Domain Glossary

- App: a user-created LLM application, persisted in `apps` (`api/models/model.py:392-468`).
- Workflow: graph-based app logic with draft and published versions (`api/controllers/console/app/workflow.py:105-150`).
- Dataset/Knowledge: RAG knowledge base stored in `datasets` (`api/models/dataset.py:166-212`).
- Tenant/Workspace: isolation boundary for users and resources (`api/controllers/console/app/wraps.py:13-18`).
- Provider: model/tool/vector/trace integration.
- Draft: editable workflow state.
- Published: deployable workflow state.
- Service hook: frontend React hook that wraps API access.
- Controller: HTTP boundary.
- Service: business logic boundary.
- Model: persistence boundary.

## Junior Socratic Checkpoint

Questions:

1. What file creates the normal backend Flask app?
2. What file registers the `/console/api` blueprint surface?
3. What providers wrap every frontend page?
4. Where does the dataset list UI get its data?
5. What validates app list query params on the backend?
6. Why is `tenant_id` important in backend queries?
7. Which test file verifies `Tag` behavior?

How to self-grade:

- Strong answers cite `api/app_factory.py:127-138`, `api/extensions/ext_blueprints.py:25-100`, `web/app/layout.tsx:61-84`, `web/service/knowledge/use-dataset.ts:83-99`, `api/controllers/console/app/app.py:66-93`, and `web/app/components/develop/__tests__/tag.spec.tsx:1-140`.
- A strong answer says tenant scoping prevents one workspace from reading another workspace's resources.

