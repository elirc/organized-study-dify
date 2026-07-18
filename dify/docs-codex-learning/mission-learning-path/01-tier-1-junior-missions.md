# Tier 1 Junior Missions

### Mission 1: The App's Heartbeat

**Tier:** Junior

**Time Estimate:** 30 minutes

**Goal:** Identify how the frontend and backend start.

**The Concept:** Every app has a heartbeat: the small set of files that bring the rest of the system to life, like a Dify workflow's start node.

**Design Intent Before You Read the Code:** The root package defines frontend scripts. The backend entry chooses between migration mode and normal runtime. The frontend root layout wraps all pages with shared providers. If these are wrong, every feature fails before its own code runs.

**Find It In The Code:** Open `package.json:1-18`, `api/app.py:32-62`, and `web/app/layout.tsx:61-84`.

```python
# api/app.py:36-54
if is_db_command():
    # Migration mode: small app for flask db commands.
    app = create_migrations_app()
else:
    # Runtime mode: full API, Socket.IO, Celery extension.
    socketio_app, flask_app = create_app()
    celery = cast("Celery", app.extensions["celery"])
```

```tsx
// web/app/layout.tsx:61-84
<JotaiProvider>
  <ThemeProvider>
    <NuqsAdapter>
      <TanstackQueryInitializer>
        <I18nServerProvider>{children}</I18nServerProvider>
      </TanstackQueryInitializer>
    </NuqsAdapter>
  </ThemeProvider>
</JotaiProvider>
```

**The Aha Moment:** Entry points do not implement product features; they create the environment every feature depends on.

**Socratic Checkpoint:** What command starts frontend dev? Why does the backend create a smaller migration app? Which provider enables React Query? Which file adds Socket.IO? What breaks if `I18nServerProvider` is missing?

**How to Self-Grade:** Strong answers cite `package.json:10-16`, `api/app_factory.py:127-138`, and `web/context/query-client.tsx:20-28`.

**Connects To:** Mission 2, because entry points are easier once you understand folder ownership.

### Mission 2: The Folder Mental Map

**Tier:** Junior

**Time Estimate:** 35 minutes

**Goal:** Build a first map of major folders and responsibilities.

**The Concept:** A codebase is like a Dify workspace: apps, datasets, providers, and workflows each need a known home.

**Design Intent Before You Read the Code:** `api/controllers` owns HTTP boundaries, `api/services` owns business logic, `api/models` owns persistence, `web/app` owns UI routes, `web/service` owns client data fetching, and `web/context` owns shared state providers.

**Find It In The Code:** Inspect folder lists, then open `api/extensions/ext_blueprints.py:25-100` and `web/app/layout.tsx:61-84`.

```python
# api/extensions/ext_blueprints.py:28-35
from controllers.console import bp as console_app_bp
from controllers.service_api import bp as service_api_bp
from controllers.web import bp as web_bp

# Each blueprint is a public doorway for a different audience.
app.register_blueprint(service_api_bp)
app.register_blueprint(web_bp)
app.register_blueprint(console_app_bp)
```

**The Aha Moment:** Folder names tell you the first guess for ownership; route registration tells you runtime ownership.

**Socratic Checkpoint:** Which folder owns user-facing pages? Which owns backend routes? Which owns database models? Which folder contains e2e tests? Which backend folder should not contain UI concerns?

**How to Self-Grade:** Strong answers distinguish `web/app`, `web/service`, `api/controllers`, `api/services`, `api/models`, and `e2e`.

**Connects To:** Mission 5, because backend route reading depends on knowing controller/service/model boundaries.

### Mission 3: TypeScript Is a Contract

**Tier:** Junior

**Time Estimate:** 40 minutes

**Goal:** Understand frontend API types as promises between layers.

**The Concept:** Types are not decoration; they are the app's API agreement, like a workflow node schema.

**Design Intent Before You Read the Code:** A query contract should define accepted inputs and expected outputs. If it drifts from backend behavior, the UI can be confidently wrong.

**Find It In The Code:** Open `web/contract/console/apps.ts:6-23`.

```ts
// web/contract/console/apps.ts:6-23
export type AppListQuery = {
  page?: number
  limit?: number
  name?: string
  mode?: AppModeEnum
  tag_ids?: string[]
}

export const appListContract = base
  .route({ path: '/apps', method: 'GET' }) // HTTP boundary
  .input(type<{ query?: AppListQuery }>()) // caller contract
  .output(type<AppListResponse>()) // response contract
```

**The Aha Moment:** TypeScript protects frontend callers, but it does not replace backend validation.

**Socratic Checkpoint:** What fields can app list accept? Which are optional? Where is backend validation? What happens if backend accepts a new field but frontend does not? What happens if frontend sends an invalid UUID?

**How to Self-Grade:** Strong answers connect `web/contract/console/apps.ts:6-23` with `api/controllers/console/app/app.py:66-93`.

**Connects To:** Mission 17, because hidden TypeScript work becomes more important in complex flows.

### Mission 4: Your First UI Component

**Tier:** Junior

**Time Estimate:** 30 minutes

**Goal:** Read a small React component and its tests.

**The Concept:** A small component is a reusable product phrase; it should have clear inputs and predictable output.

**Design Intent Before You Read the Code:** `Tag` converts an HTTP method-like string into visual styling. Poor design would scatter method colors across many files.

**Find It In The Code:** Open `web/app/components/develop/tag.tsx:4-60` and `web/app/components/develop/__tests__/tag.spec.tsx:1-140`.

```tsx
// web/app/components/develop/tag.tsx:36-60
const valueColorMap = { get: 'emerald', post: 'sky', put: 'amber', delete: 'rose' }

export function Tag({ children, variant = 'medium', color = valueColorMap[children.toLowerCase()] ?? 'emerald' }: ITagProps) {
  // cn combines base typography, variant styles, and color styles.
  return <span className={cn('font-mono text-[0.625rem] leading-6 font-semibold', variantStyles[variant], colorStyles[color]![variant])}>{children}</span>
}
```

**The Aha Moment:** Even simple components encode product vocabulary and reviewable behavior.

**Socratic Checkpoint:** What is the default color for `POST`? What happens for `UNKNOWN`? What element renders? What tests prove this? What runtime risk does `colorStyles[color]!` hide?

**How to Self-Grade:** Strong answers cite tests at `web/app/components/develop/__tests__/tag.spec.tsx:18-60`.

**Connects To:** Mission 6, because props are contracts.

### Mission 5: Your First Backend Route

**Tier:** Junior

**Time Estimate:** 45 minutes

**Goal:** Read one backend route from decorators to service call.

**The Concept:** A controller is a checkpoint gate: setup, auth, validation, permission, delegation.

**Design Intent Before You Read the Code:** The app list endpoint should not know SQL details; it should validate inputs and call `AppService`.

**Find It In The Code:** Open `api/controllers/console/app/app.py:459-532`.

```python
# api/controllers/console/app/app.py:459-485
@console_ns.route("/apps")
class AppListApi(Resource):
    @setup_required
    @login_required
    @account_initialization_required
    @enterprise_license_required
    def get(self):
        current_user, current_tenant_id = current_account_with_tenant()
        args = AppListQuery.model_validate(_normalize_app_list_query_args(request.args))
        app_pagination = AppService().get_paginate_apps(current_user.id, current_tenant_id, params)
```

**The Aha Moment:** The controller's main job is to protect the boundary and hand work to the right owner.

**Socratic Checkpoint:** Which decorators run before the method? Where is tenant id obtained? What validates query args? What service is called? What response model is returned?

**How to Self-Grade:** Strong answers mention `current_account_with_tenant`, `AppListQuery`, and `AppPagination`.

**Connects To:** Mission 13, because decorators form a middleware/boundary chain.

### Mission 6: Props, Inputs, Or Parameters Are Contracts

**Tier:** Junior

**Time Estimate:** 35 minutes

**Goal:** Compare frontend props and backend payload models.

**The Concept:** Every boundary is a contract, whether it is a React prop, HTTP query, or Pydantic payload.

**Design Intent Before You Read the Code:** Dataset components should accept display filters, while backend payloads should validate product rules.

**Find It In The Code:** Open `web/app/components/datasets/list/datasets.tsx:11-38` and `api/controllers/console/datasets/datasets.py:130-149`.

```tsx
// web/app/components/datasets/list/datasets.tsx:11-23
type Props = {
  tags: string[]
  keywords: string
  includeAll: boolean
}
```

```python
# api/controllers/console/datasets/datasets.py:130-149
class DatasetCreatePayload(BaseModel):
    name: str = Field(..., min_length=1, max_length=40)
    provider: str = "vendor"

    @field_validator("provider")
    def validate_provider(cls, value):
        if value not in Dataset.PROVIDER_LIST:
            raise ValueError("Invalid provider.")
```

**The Aha Moment:** Frontend contracts guide usage; backend contracts enforce correctness.

**Socratic Checkpoint:** Which dataset prop changes API data? Which payload field is required? Where are provider values defined? What should be tested if `provider` changes? Why are frontend and backend contracts different?

**How to Self-Grade:** Strong answers cite `api/models/dataset.py:174-176`.

**Connects To:** Mission 12, because API contracts combine frontend and backend boundaries.

### Mission 7: Following Data Into The App

**Tier:** Junior

**Time Estimate:** 50 minutes

**Goal:** Trace dataset list data from UI filter to backend response.

**The Concept:** Data fetching is like retrieval in a RAG pipeline: query shape controls what comes back.

**Design Intent Before You Read the Code:** The UI debounces filter changes, the hook builds a stable query key, and the backend applies validated filters.

**Find It In The Code:** Open `web/app/components/datasets/list/index.tsx:35-52`, `web/service/knowledge/use-dataset.ts:83-99`, and `api/controllers/console/datasets/datasets.py:289-379`.

```ts
// web/service/knowledge/use-dataset.ts:83-99
return useInfiniteQuery({
  queryKey: [...datasetListQueryKey, initialPage, tag_ids, limit, include_all, keyword],
  queryFn: ({ pageParam = 1 }) => {
    const urlParams = qs.stringify({ tag_ids, limit, include_all, keyword, page: pageParam }, { indices: false })
    return get<DataSetListResponse>(`/datasets?${urlParams}`)
  },
})
```

**The Aha Moment:** A query key must include every input that can change the result.

**Socratic Checkpoint:** Which state is debounced? What fields enter the query key? What backend class validates query params? How is `has_more` computed? What bug appears if `keyword` is missing from the key?

**How to Self-Grade:** Strong answers cite `web/app/components/datasets/list/index.tsx:35-52` and `api/controllers/console/datasets/datasets.py:372-379`.

**Connects To:** Mission 9, because state needs a home and a reason.

### Mission 8: Navigation And Routing Are The App's Skeleton

**Tier:** Junior

**Time Estimate:** 35 minutes

**Goal:** Understand frontend route groups and backend blueprints.

**The Concept:** Routes are the skeleton: they do not do all the work, but they determine where movement is possible.

**Design Intent Before You Read the Code:** Frontend route groups organize layouts without always changing URL segments. Backend blueprints expose surfaces for console, webapp, service API, files, inner API, MCP, and triggers.

**Find It In The Code:** Open `web/app/(commonLayout)/layout.tsx`, `web/app/(shareLayout)/layout.tsx`, and `api/extensions/ext_blueprints.py:25-100`.

```python
# api/extensions/ext_blueprints.py:42-100
app.register_blueprint(service_api_bp)
app.register_blueprint(web_bp)
app.register_blueprint(console_app_bp)
app.register_blueprint(files_bp)
app.register_blueprint(inner_api_bp)
app.register_blueprint(mcp_bp)
app.register_blueprint(trigger_bp)
```

**The Aha Moment:** Route structure reveals audience boundaries.

**Socratic Checkpoint:** Which backend blueprint is for console users? Which is for webapp usage? Why do trigger routes have special CORS? What do route groups organize? What file registers all backend blueprints?

**How to Self-Grade:** Strong answers cite `api/extensions/ext_blueprints.py:36-100`.

**Connects To:** Mission 14, because end-to-end tracing crosses both frontend and backend routing.

