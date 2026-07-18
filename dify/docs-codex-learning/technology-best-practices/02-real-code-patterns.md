# Real Code Patterns

## Technology: Next.js And React

### Where It Is Used

The web app uses React components under `web/app/`, with Next.js App Router layout composition at `web/app/layout.tsx`.

### Important Files

- `web/app/layout.tsx:29-91`: root app shell and provider composition.
- `web/app/components/develop/tag.tsx:4-60`: small reusable UI component.
- `web/app/components/datasets/list/index.tsx:25-108`: feature-level component with local state and service hooks.
- `web/app/components/workflow-app/index.tsx:40-155`: complex editor shell.

### Best-Practice Example 1: Root Provider Composition

File: `web/app/layout.tsx:61-84`

```tsx
<Provider>
  {/* Jotai provider creates the atom store for descendants. */}
  <ThemeProvider>
    {/* Theme is app-wide infrastructure, so it belongs near the root. */}
    <NuqsAdapter>
      <TanstackQueryInitializer>
        {/* React Query must wrap components that call data hooks. */}
        <I18nServerProvider locale={locale}>
          {/* i18n is wired above UI components that render translated strings. */}
          {children}
          <ToastHost />
          <TooltipProvider />
          <CheckUpdate />
        </I18nServerProvider>
      </TanstackQueryInitializer>
    </NuqsAdapter>
  </ThemeProvider>
</Provider>
```

This is a good pattern because cross-cutting UI infrastructure is centralized. A junior developer may only see "a stack of wrappers." A stronger engineer sees dependency ordering: query hooks need a query client, translated components need i18n context, and global UI helpers need to live outside feature screens.

### Best-Practice Example 2: Small Component With Explicit Variants

File: `web/app/components/develop/tag.tsx:4-60`

The `Tag` component keeps presentation decisions in one place: color classes, size classes, close behavior, and disabled behavior. That makes the component easy to test in `web/app/components/develop/__tests__/tag.spec.tsx:1-140`.

A junior should notice the prop API. A mid-level engineer should ask whether a new tag behavior belongs in this shared component or in a feature wrapper.

### Best-Practice Example 3: Feature Component Delegates Fetching

File: `web/app/components/datasets/list/index.tsx:25-108`

This component owns UI state such as search text, tag filtering, and toggles, while `useDatasetList` owns data fetching. That separation keeps rendering and server-state behavior from becoming one tangled function.

### Weak Or Risky Pattern, If Present

The workflow editor in `web/app/components/workflow-app/index.tsx:40-155` is necessarily dense because it coordinates graph state, workflow data, triggers, and provider setup. Dense shell components are not automatically bad, but they need strong boundaries around hooks and stores. When editing this file, look for opportunities to preserve existing separation rather than pushing more business logic into the shell.

### Anti-Patterns To Avoid

- Do not hardcode user-facing strings in new frontend code; this repo expects i18n through `web/i18n/en-US/`.
- Do not fetch directly inside presentation-only leaf components if an existing service hook owns that data.
- Do not add another global provider until you can explain why React Query, Zustand, Jotai, or local state is insufficient.

### Interview Talking Points

- "The app uses Next.js App Router with provider composition at the root layout."
- "Feature screens generally combine local UI state with service hooks rather than embedding raw fetch calls."
- "Complex workflow editing state is separated into hooks and stores because server state and graph editing state have different lifecycles."

### Mini Practice Tasks

- Explain `web/app/layout.tsx:61-84` out loud without naming framework buzzwords.
- Add a test case to the `Tag` test file in a local branch and discard it after review.
- Trace one prop from `web/app/components/datasets/list/index.tsx:25-108` into a child component.

## Technology: TypeScript

### Where It Is Used

TypeScript is the frontend's primary safety layer across components, hooks, service clients, and contracts.

### Important Files

- `web/app/components/datasets/list/datasets.tsx:11-38`: component props and hook result usage.
- `web/service/base.ts:41-132`: stream callback types.
- `web/contract/console/apps.ts:6-47`: typed API contract definitions.
- `web/service/fetch.ts:21-24`: weaker shared request options.

### Best-Practice Example 1: Contract Types Near API Shape

File: `web/contract/console/apps.ts:6-47`

This file describes app-list query shape, delete route shape, and workflow online-user route shape near the contract declaration. The value is not just autocomplete. It makes frontend/backend expectations reviewable.

### Best-Practice Example 2: Stream Callback Types

File: `web/service/base.ts:41-132`

The streaming types describe callback payloads for SSE-style responses. A junior may read them as ceremony. A mid-level engineer sees a contract for long-running interactions where runtime debugging can otherwise be painful.

### Weak Or Risky Pattern, If Present

File: `web/service/fetch.ts:21-24`

```ts
export type FetchOptionType = Omit<Options, 'body'> & {
  body?: BodyInit | Record<string, any> | null
}
```

The `any` keeps the wrapper permissive for many payload shapes, but it means callers can pass objects that do not match an endpoint. A safer pattern would use generic payload types at endpoint-specific boundaries while keeping the low-level wrapper broad.

### Anti-Patterns To Avoid

- Do not introduce `any` in a feature file just to silence a compiler error.
- Do not type server-state as UI state. Query results, form state, and local workflow graph state have different lifecycles.
- Do not let generated and manual contracts drift without checking backend route shape.

### Interview Talking Points

- "TypeScript is strongest where endpoint contracts and component props are explicit."
- "The project still has pragmatic `any` in shared infrastructure and workflow graph areas, which is a trade-off worth reviewing carefully."

### Mini Practice Tasks

- Replace a local `any` in a throwaway branch with a narrow interface, then inspect what errors become clearer.
- Compare `web/contract/console/apps.ts:6-47` with `api/controllers/console/app/app.py:459-561`.

## Technology: React Query

### Where It Is Used

React Query manages server state for frontend features, including dataset lists and oRPC-backed queries.

### Important Files

- `web/context/query-client.tsx:1-28`
- `web/service/knowledge/use-dataset.ts:28-104`
- `web/service/client.ts:87-137`

### Best-Practice Example 1: Query Client Provider

File: `web/context/query-client.tsx:1-28`

The query client is initialized once and provided to the React tree. This prevents every feature from inventing its own cache.

### Best-Practice Example 2: Dataset Query Key

File: `web/service/knowledge/use-dataset.ts:28-104`

The dataset list hook normalizes query params and keeps the query key near the function that fetches the list. That is a maintainable pattern because changing pagination or filters forces you to think about cache identity.

### Best-Practice Example 3: oRPC Query Defaults

File: `web/service/client.ts:87-137`

This file defines query defaults for tag-based invalidation and caching. A senior engineer should notice that cache invalidation belongs near the API client layer when many features consume the same contracts.

### Weak Or Risky Pattern, If Present

React Query can be misused if developers place editable local UI state into query cache. In this repo, workflow graph editing uses Zustand instead, which is the healthier boundary for local mutable editor state.

### Anti-Patterns To Avoid

- Do not create cache keys from unstable objects unless they are normalized first.
- Do not call raw fetch from a component when an existing service hook already owns the data.
- Do not invalidate broad query groups when a narrow key would update the screen.

### Interview Talking Points

- "React Query handles server state; Zustand handles workflow editor state."
- "Dataset list fetching shows how query keys, params, and pagination work together."

### Mini Practice Tasks

- Trace how changing `keyword` in `web/app/components/datasets/list/index.tsx:35-52` changes the query in `web/service/knowledge/use-dataset.ts:28-104`.
- Find one mutation hook and identify what should be invalidated after success.

## Technology: Zustand And Client State

### Where It Is Used

Zustand appears in workflow editor state, where local graph state is too interactive and mutable for server-state tools.

### Important Files

- `web/app/components/workflow-app/store/workflow/workflow-slice.ts:3-35`
- `web/app/components/workflow-app/index.tsx:40-155`
- `web/app/components/workflow-app/hooks/use-workflow-run.ts:99-516`

### Best-Practice Example 1: State Slice Shape

File: `web/app/components/workflow-app/store/workflow/workflow-slice.ts:3-35`

The slice groups workflow-related values and setters. This makes it easier to reason about editor state than scattering many `useState` calls across the workflow shell.

### Best-Practice Example 2: Hook Coordinates Side Effects

File: `web/app/components/workflow-app/hooks/use-workflow-run.ts:191-441`

`handleRun` coordinates graph validation, draft synchronization, run state, callbacks, and SSE execution. Keeping this in a hook lets the component use workflow execution as a capability instead of owning every step inline.

### Weak Or Risky Pattern, If Present

The workflow slice uses broad types in `web/app/components/workflow-app/store/workflow/workflow-slice.ts:3-35`. This reduces friction while the graph model evolves, but it makes breaking changes harder to catch at compile time.

### Anti-Patterns To Avoid

- Do not put server-fetched entities into Zustand unless they are truly editor-local copies.
- Do not update workflow graph state in two unrelated stores.
- Do not bypass `handleRun`-style orchestration for workflow execution without understanding draft sync.

### Interview Talking Points

- "Zustand is used where the frontend has rich local interaction state."
- "Workflow execution is coordinated by hooks because it crosses validation, persistence, streaming, and UI state."

### Mini Practice Tasks

- Draw the state transitions inside `handleRun` from idle to running to stopped.
- Identify which workflow values are local UI state and which come from backend persistence.

## Technology: ky HTTP Wrapper And oRPC Contracts

### Where It Is Used

The frontend uses a ky wrapper for general HTTP behavior and oRPC/OpenAPI contracts for typed API access.

### Important Files

- `web/service/fetch.ts:103-277`
- `web/service/base.ts:181-187`
- `web/service/client.ts:35-90`
- `web/contract/router.ts:56-120`
- `web/contract/console/apps.ts:6-47`

### Best-Practice Example 1: Request Normalization

File: `web/service/fetch.ts:137-249`

The wrapper adds CSRF headers, handles auth/passport headers, serializes bodies, and normalizes responses. This is a good boundary because request infrastructure should not be repeated in every feature hook.

### Best-Practice Example 2: Contract Router

File: `web/contract/router.ts:56-120`

The router gathers marketplace and console contracts. The important engineering lesson is that API shape becomes searchable and reviewable instead of living only in ad hoc strings.

### Weak Or Risky Pattern, If Present

File: `web/contract/router.ts:65-69`

The comments state generated community OpenAPI files are currently incomplete. This mixed state is acceptable during migration, but reviewers should be alert for drift between backend routes and frontend contract definitions.

### Anti-Patterns To Avoid

- Do not duplicate base URL logic in a feature.
- Do not add a new endpoint call without checking whether a contract or service hook already exists.
- Do not swallow HTTP errors in a component if the shared wrapper or hook layer should normalize them.

### Interview Talking Points

- "The project uses a shared ky wrapper for HTTP mechanics and oRPC contracts for typed endpoint shape."
- "Contract migration is incomplete, so reviewers must compare frontend contracts with backend controllers."

### Mini Practice Tasks

- Compare `web/contract/console/apps.ts:6-47` to the backend `/apps` resource.
- Trace how a CSRF token gets added in `web/service/fetch.ts:181-185`.

## Technology: Flask And Flask-RESTX

### Where It Is Used

The backend API is a Flask application with Flask-RESTX resources grouped under blueprints and namespaces.

### Important Files

- `api/app.py:32-62`
- `api/app_factory.py:49-138`
- `api/extensions/ext_blueprints.py:25-100`
- `api/controllers/console/app/app.py:459-561`
- `api/controllers/console/datasets/datasets.py:289-450`

### Best-Practice Example 1: App Factory

File: `api/app_factory.py:49-138`

The factory creates and configures the Flask app, registers hooks, loads extensions, and returns an app object. This pattern makes initialization explicit and testable.

### Best-Practice Example 2: Blueprint Registration

File: `api/extensions/ext_blueprints.py:25-100`

Blueprint registration groups public service API, console API, web routes, file routes, internal API, MCP, and trigger routes. This helps you reason about which route family owns a behavior.

### Best-Practice Example 3: Resource Class

File: `api/controllers/console/app/app.py:459-561`

The `/apps` resource handles list and create operations through decorators, request parsing, Pydantic validation, service calls, and response marshalling.

### Weak Or Risky Pattern, If Present

Flask decorators are powerful but can hide important behavior. For example, auth, setup checks, and tenant loading can be above the method body. A reviewer must read decorators before reading business logic.

### Anti-Patterns To Avoid

- Do not put business logic directly in controller methods when a service already owns the domain behavior.
- Do not add a route without tenant/auth decorators when the data is workspace-specific.
- Do not parse raw request data repeatedly if a Pydantic model can validate it once.

### Interview Talking Points

- "Backend requests enter through Flask, are grouped by blueprints, validated at controller boundaries, then delegated to services."
- "Decorators form part of the backend security and tenant model."

### Mini Practice Tasks

- Trace the `/apps` GET route from blueprint registration to service method.
- For one route, list every decorator and explain the failure case each protects.

## Technology: Pydantic

### Where It Is Used

Pydantic validates backend request bodies and query parameters before service logic receives data.

### Important Files

- `api/controllers/console/app/app.py:66-131`
- `api/controllers/console/datasets/datasets.py:130-207`

### Best-Practice Example 1: App List Query

File: `api/controllers/console/app/app.py:66-93`

`AppListQuery` constrains pagination, mode filters, name search, and tag IDs. Its tag validator handles comma-separated string input and converts it into UUID values.

### Best-Practice Example 2: Dataset Create Payload

File: `api/controllers/console/datasets/datasets.py:130-149`

Dataset creation validates name and permission-like fields near the route boundary. This keeps invalid user input away from service code.

### Weak Or Risky Pattern, If Present

Pydantic validation is only as useful as the routes that use it. If a future route accepts raw dictionaries and passes them into services, it weakens the boundary. Review new controllers for explicit validation.

### Anti-Patterns To Avoid

- Do not validate the same field in the UI, controller, and service with three inconsistent rules.
- Do not treat Pydantic models as database models; they are request/response boundaries.
- Do not skip validators for IDs that are later used in queries.

### Interview Talking Points

- "Pydantic models turn raw HTTP input into typed application input."
- "Validators are especially useful for filters and IDs that arrive as strings."

### Mini Practice Tasks

- Explain why `tag_ids` normalization belongs at the boundary.
- Find one field where frontend validation should mirror backend validation.

## Technology: SQLAlchemy

### Where It Is Used

SQLAlchemy models define persisted backend entities such as apps, datasets, accounts, tenants, workflows, and related domain records.

### Important Files

- `api/models/base.py:14-104`
- `api/models/model.py:392-490`
- `api/models/dataset.py:166-220`
- `api/services/app_service.py:56-203`

### Best-Practice Example 1: Base Mixins

File: `api/models/base.py:14-104`

Base model mixins centralize common ID, timestamp, and UUID behavior. This avoids repeated boilerplate across every model.

### Best-Practice Example 2: App Model

File: `api/models/model.py:392-490`

The `App` model captures tenant ownership, mode, icon metadata, timestamps, and relationships/properties used by services and controllers.

### Best-Practice Example 3: Service Query Boundary

File: `api/services/app_service.py:56-100`

`get_paginate_apps` composes tenant filters, mode filters, name filters, tag filters, and pagination in the service layer. That is cleaner than building queries inside controllers.

### Weak Or Risky Pattern, If Present

Model properties such as `api/models/model.py:426-468` and `api/models/dataset.py:213-220` can trigger database queries when accessed. This is convenient, but in list responses it can create hidden N+1 query problems.

### Anti-Patterns To Avoid

- Do not add hidden database work to a property without considering list endpoints.
- Do not query tenant-owned data without tenant filters.
- Do not put request parsing into model classes.

### Interview Talking Points

- "SQLAlchemy models express persistence, while services compose business queries."
- "Hidden query properties are a trade-off between convenience and performance."

### Mini Practice Tasks

- Identify one model property that queries the database and explain where it is safe or unsafe.
- Trace an app list filter from query parameter to SQLAlchemy query.

## Technology: Testing

### Where It Is Used

Frontend unit tests use Vitest and React Testing Library. Backend unit tests use pytest. E2E tests use Cucumber and Playwright.

### Important Files

- `web/app/components/develop/__tests__/tag.spec.tsx:1-140`
- `web/app/components/workflow-app/hooks/__tests__/use-workflow-run.spec.ts:1-180`
- `api/tests/unit_tests/tasks/test_workflow_execute_task.py:73-160`
- `e2e/package.json`

### Best-Practice Example 1: Component Behavior Test

File: `web/app/components/develop/__tests__/tag.spec.tsx:1-140`

The `Tag` tests assert visible behavior: rendering, close interactions, disabled state, and styling-relevant variants. This is stronger than snapshot-only testing.

### Best-Practice Example 2: Hook Test With Mocks

File: `web/app/components/workflow-app/hooks/__tests__/use-workflow-run.spec.ts:1-180`

The hook test mocks dependencies around workflow execution. This is important because workflow running has many side effects and would be expensive to test only through the browser.

### Best-Practice Example 3: Backend Task Test

File: `api/tests/unit_tests/tasks/test_workflow_execute_task.py:73-160`

The backend task test validates behavior around workflow execution task handling. This is the right testing level for logic that should not require a full browser.

### Weak Or Risky Pattern, If Present

Setup docs indicate integration tests are CI-oriented, and local integration test setup may require services and secrets. That is normal for a large app, but it means local confidence often depends on unit tests plus targeted manual verification.

### Anti-Patterns To Avoid

- Do not test implementation details when visible behavior is enough.
- Do not skip tests for risky hooks just because they are "only frontend."
- Do not make backend unit tests require live external providers unless the test is explicitly integration-level.

### Interview Talking Points

- "The repo uses different test levels for components, hooks, backend tasks, and browser journeys."
- "Workflow execution deserves focused tests because it crosses state, validation, persistence, and streaming."

### Mini Practice Tasks

- Write a documentation-only test plan for the dataset API toggle story.
- Read one hook test and identify which dependencies are mocked and why.

