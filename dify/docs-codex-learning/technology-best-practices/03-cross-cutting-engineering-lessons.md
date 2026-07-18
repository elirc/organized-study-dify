# Cross-Cutting Engineering Lessons

## Separation Of Concerns

The concept: each layer should own one kind of decision. UI components render and collect intent. Hooks fetch and mutate data. HTTP wrappers normalize transport behavior. Controllers validate requests. Services perform business operations. Models represent persistence.

Where it appears:

- UI state and filters: `web/app/components/datasets/list/index.tsx:25-108`
- Fetching and cache keys: `web/service/knowledge/use-dataset.ts:28-104`
- HTTP normalization: `web/service/fetch.ts:137-249`
- Backend route: `api/controllers/console/datasets/datasets.py:289-450`
- Persistence model: `api/models/dataset.py:166-220`

Junior developers often miss that "the dataset list" is not one file. Senior engineers notice whether a proposed change is placed at the layer where the decision belongs.

Practice exercise: choose one dataset UI behavior and name the file responsible for rendering, fetching, validation, and persistence.

## Type Safety

The concept: types make boundaries explicit. A type should tell you what a caller may send, what a function promises, and what shape a response has.

Where it appears:

- Component props: `web/app/components/datasets/list/datasets.tsx:11-38`
- API contracts: `web/contract/console/apps.ts:6-47`
- Stream callback types: `web/service/base.ts:41-132`
- Weak shared payload type: `web/service/fetch.ts:21-24`

Junior developers often see TypeScript as autocomplete. Senior engineers ask where the compiler stops protecting the system, especially around `any`.

Practice exercise: find one frontend service function and write down the input type, output type, and runtime failure modes.

## Validation Boundaries

The concept: validation belongs at boundaries where untrusted input enters the system. Frontend validation improves experience, but backend validation protects data integrity.

Where it appears:

- App list query validation: `api/controllers/console/app/app.py:66-93`
- App create/update payloads: `api/controllers/console/app/app.py:115-131`
- Dataset create validation: `api/controllers/console/datasets/datasets.py:130-149`
- Dataset list query validation: `api/controllers/console/datasets/datasets.py:199-207`

Junior developers often duplicate validation inconsistently. Senior engineers look for a single authoritative backend boundary, then mirror it in the UI for better feedback.

Practice exercise: pick one backend Pydantic field and explain what bad request it prevents.

## API Design

The concept: API design is not only route URLs. It includes auth decorators, request schemas, response shape, error behavior, and frontend cache behavior.

Where it appears:

- `/apps` route: `api/controllers/console/app/app.py:459-561`
- `/datasets` routes: `api/controllers/console/datasets/datasets.py:289-450`
- frontend app contract: `web/contract/console/apps.ts:6-47`
- contract router gap note: `web/contract/router.ts:65-69`

Junior developers often compare only method names. Senior engineers compare the route, schema, auth, service call, response model, and frontend caller together.

Practice exercise: compare the app-list contract with the backend app-list route and list any fields that need extra verification.

## State Management

The concept: state has categories. Server state belongs in React Query. Rich local editor state belongs in stores or local component state. App-wide context belongs near providers.

Where it appears:

- React Query provider: `web/context/query-client.tsx:1-28`
- App context provider: `web/context/app-context-provider.tsx:30-159`
- Dataset server state: `web/service/knowledge/use-dataset.ts:28-104`
- Workflow local state: `web/app/components/workflow-app/store/workflow/workflow-slice.ts:3-35`

Junior developers often ask, "Where do I put state?" Senior engineers ask, "Who owns freshness, mutation, persistence, and reset behavior?"

Practice exercise: classify five values in the workflow editor as server state, local UI state, or app context.

## Data Fetching

The concept: data fetching is a chain: UI intent, hook params, cache key, request wrapper, backend validation, service query, response normalization, UI render.

Where it appears:

- dataset filters: `web/app/components/datasets/list/index.tsx:35-52`
- dataset hook: `web/service/knowledge/use-dataset.ts:28-104`
- request wrapper: `web/service/fetch.ts:137-249`
- backend dataset list route: `api/controllers/console/datasets/datasets.py:289-379`

Junior developers often debug the visible component only. Senior engineers trace the full chain and ask where the data changed shape.

Practice exercise: change a filter value mentally and predict every layer it touches.

## Database Access

The concept: persistence code should preserve tenant boundaries, minimize hidden queries, and keep query composition near business logic.

Where it appears:

- App service query: `api/services/app_service.py:56-100`
- App model: `api/models/model.py:392-490`
- Dataset model: `api/models/dataset.py:166-220`
- hidden dataset count property: `api/models/dataset.py:213-220`

Junior developers often think an ORM property is free. Senior engineers ask what SQL might run and how often.

Practice exercise: identify one list endpoint and check whether its response serialization may touch query-backed properties.

## Error Handling

The concept: errors should be normalized at boundaries and surfaced at the right level. A component should not need to know every low-level HTTP detail.

Where it appears:

- fetch response normalization: `web/service/fetch.ts:137-249`
- backend service commits and exceptions around app creation: `api/services/app_service.py:102-203`
- workflow run abort/stop behavior: `web/app/components/workflow-app/hooks/use-workflow-run.ts:444-516`

Junior developers often catch errors too early. Senior engineers decide whether an error is transport, validation, domain, permission, or user-action related.

Practice exercise: describe how a failed workflow stop should be handled in UI, hook, service call, and backend route.

## Testing Strategy

The concept: test at the cheapest level that gives confidence. Component behavior belongs in component tests, hook orchestration in hook tests, backend task logic in unit tests, and full journeys in E2E tests.

Where it appears:

- component test: `web/app/components/develop/__tests__/tag.spec.tsx:1-140`
- hook test: `web/app/components/workflow-app/hooks/__tests__/use-workflow-run.spec.ts:1-180`
- backend task test: `api/tests/unit_tests/tasks/test_workflow_execute_task.py:73-160`
- E2E tooling: `e2e/package.json`

Junior developers often write either no tests or only browser tests. Senior engineers use a test pyramid and reserve slow tests for cross-layer confidence.

Practice exercise: for a dataset API toggle, choose one unit test, one component test, and one manual verification step.

## Maintainability

The concept: maintainable code makes change paths obvious. You should be able to find the owner of a behavior quickly.

Where it appears:

- blueprint route families: `api/extensions/ext_blueprints.py:25-100`
- service boundary: `api/services/app_service.py:56-203`
- service hook boundary: `web/service/knowledge/use-dataset.ts:28-187`

Junior developers often optimize for "where can I make it work?" Senior engineers optimize for "where will the next maintainer expect this?"

Practice exercise: before editing a behavior, write down the current owner and the reason it owns that behavior.

## Scalability

The concept: scalability includes backend load, frontend render cost, cache behavior, async execution, and database query count.

Where it appears:

- Celery/task testing: `api/tests/unit_tests/tasks/test_workflow_execute_task.py:73-160`
- workflow run streaming: `web/app/components/workflow-app/hooks/use-workflow-run.ts:191-441`
- hidden model queries: `api/models/model.py:426-468`, `api/models/dataset.py:213-220`

Junior developers often think scalability only means more servers. Senior engineers look for synchronous work in request paths, N+1 queries, broad invalidations, and expensive re-renders.

Practice exercise: explain why workflow execution uses streaming/task concepts instead of a simple blocking request.

## Security Posture

The concept: security lives in layers: auth, tenant scope, CSRF, route decorators, permission checks, and safe data access.

Where it appears:

- login and CSRF checks: `api/libs/login.py:85-100`
- current account and tenant: `api/libs/login.py:38-48`
- tenant app loading: `api/controllers/console/app/wraps.py:13-145`
- frontend CSRF header: `web/service/fetch.ts:181-185`
- billing/resource wrapper: `api/controllers/console/wraps.py:94-133`

Junior developers often check only whether a route "requires login." Senior engineers check tenant ownership, CSRF, role/resource limits, and data leaks in response shape.

Practice exercise: for one new route idea, list the auth, tenant, and validation checks before writing code.

## Performance

The concept: performance problems often come from repeated work hidden behind pleasant abstractions.

Where it appears:

- model properties that query: `api/models/model.py:426-468`, `api/models/dataset.py:213-220`
- infinite dataset list rendering: `web/app/components/datasets/list/datasets.tsx:43-58`
- workflow execution side effects: `web/app/components/workflow-app/hooks/use-workflow-run.ts:191-441`

Junior developers often profile only the browser. Senior engineers inspect query count, network waterfall, cache keys, render loops, and async task boundaries.

Practice exercise: sketch how you would detect an N+1 query in the dataset list without changing production code.

## Developer Experience

The concept: good DX makes correct behavior easy and risky behavior visible.

Where it appears:

- root scripts: `package.json`
- frontend scripts: `web/package.json`
- backend Makefile commands: `Makefile:66-105`
- setup process: `Makefile:17-46`

Junior developers often memorize commands. Senior engineers document command ownership and failure modes.

Practice exercise: write a local setup checklist and mark each command as frontend, backend, middleware, or test-related.

## Configuration Management

The concept: configuration should be explicit, environment-aware, and reproducible.

Where it appears:

- env templates: `api/.env.example`, `web/.env.example`, `docker/.env.example`
- setup copies: `Makefile:17-46`
- app config loading: `api/app_factory.py:49-124`

Junior developers often copy `.env` files without reading them. Senior engineers identify which config values are required for local development, tests, middleware, and external providers.

Practice exercise: list which env files are needed for frontend-only work versus full-stack backend work.

## Dependency Management

The concept: dependencies are architectural commitments. They shape testing, deployment, security, and future upgrades.

Where it appears:

- root package manager pin: `package.json`
- frontend dependencies: `web/package.json`
- backend dependencies: `api/pyproject.toml`
- E2E dependencies: `e2e/package.json`

Junior developers often add dependencies for convenience. Senior engineers ask whether the repo already has a local pattern or helper that solves the problem.

Practice exercise: before adding a UI library, search for an existing component and explain why it does or does not fit.

