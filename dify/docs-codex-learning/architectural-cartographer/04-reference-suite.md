# Reference Suite

## Doc 1: Junior Onboarding Guide

Start here:

- Read `package.json:1-18` and `web/package.json:25-43` to understand scripts.
- Read `Makefile:17-46` to reconstruct setup.
- Read `api/app.py:32-62` and `web/app/layout.tsx:61-84`.
- Trace one simple component: `web/app/components/develop/tag.tsx:43-60`.
- Run or inspect tests for that component: `web/app/components/develop/__tests__/tag.spec.tsx:1-140`.

Day-one questions to answer:

- How does the backend start?
- How does the frontend get global providers?
- What is a tenant/workspace?
- Which layer validates user input?
- Which layer writes to the database?

## Doc 2: Mid-Level Architecture Guide

System design:

- Frontend pages live under `web/app`.
- Global frontend state/context starts in `web/app/layout.tsx:61-84` and `web/context/app-context-provider.tsx:30-159`.
- Data fetching uses React Query, manual wrappers, and partial oRPC contracts (`web/service/client.ts:68-137`).
- Backend routes are registered through blueprints (`api/extensions/ext_blueprints.py:25-100`).
- Controllers validate and delegate (`api/controllers/console/app/app.py:469-561`).
- Services own core logic (`api/services/app_service.py:56-203`).
- SQLAlchemy models persist domain objects (`api/models/model.py:392-468`).

Data-flow rule: trace UI state, query key, request path, controller payload, service call, database query, response DTO, and cache update.

## Doc 3: Senior Ownership Guide

Technical debt register:

- Hidden DB queries in model properties: `api/models/model.py:441-468`, `api/models/dataset.py:213-220`.
- Hybrid contract system: `web/contract/router.ts:65-69`.
- `any` in fetch/store boundaries: `web/service/fetch.ts:21-24`, `web/app/components/workflow-app/store/workflow/workflow-slice.ts:10-11`.
- High-complexity workflow orchestration: `web/app/components/workflow-app/hooks/use-workflow-run.ts:191-441`.

Upgrade path:

1. Add tests before refactors.
2. Batch database lookups in list endpoints.
3. Make API contract ownership explicit.
4. Gradually narrow weak TypeScript boundaries.
5. Keep setup docs and Makefile aligned.

## Doc 4: Code Review Guide

Review in this order:

1. Boundary: route, component, hook, service, model.
2. Tenant/auth: see `api/libs/login.py:85-100` and `api/controllers/console/app/wraps.py:13-18`.
3. Validation: Pydantic models such as `api/controllers/console/app/app.py:66-93`.
4. Data fetching: query keys like `web/service/knowledge/use-dataset.ts:83-99`.
5. Cache invalidation: `web/service/knowledge/use-dataset.ts:102-104`.
6. Persistence: service filters/commits such as `api/services/app_service.py:65-100`, `api/services/app_service.py:180-203`.
7. Tests: behavior-focused tests from `web/docs/test.md:40-52`.

Example review question: "Does this new endpoint load by id and tenant, or only by id?"

## Doc 5: Debugging Guide

Frontend data bug:

1. Find component state.
2. Find hook.
3. Inspect query key.
4. Inspect request path.
5. Inspect response shape.

Example dataset path:

- UI filters: `web/app/components/datasets/list/index.tsx:35-52`.
- Infinite list call: `web/app/components/datasets/list/datasets.tsx:26-38`.
- Hook query key and URL: `web/service/knowledge/use-dataset.ts:83-99`.
- Backend route: `api/controllers/console/datasets/datasets.py:289-379`.

Backend permission bug:

1. Check decorators.
2. Check tenant-aware load helper.
3. Check service filters.
4. Check model relationship/property side effects.

Workflow streaming bug:

- Frontend run orchestration: `web/app/components/workflow-app/hooks/use-workflow-run.ts:191-441`.
- Backend draft run route: `api/controllers/console/app/workflow.py:697-733`.
- Stop path: `api/controllers/console/app/workflow.py:735-759`.

## Doc 6: Change Playbook

Safe feature change:

1. Find a similar feature.
2. Copy the layer structure, not the code blindly.
3. Update frontend type/contract or service hook.
4. Update UI with i18n strings under `web/i18n/en-US/`.
5. Update backend Pydantic model.
6. Keep controller thin.
7. Put business logic in service.
8. Preserve tenant scoping.
9. Add or update tests.
10. Run focused checks.

For an app-list change, inspect:

- `web/contract/console/apps.ts:6-23`
- `web/service/client.ts:87-137`
- `api/controllers/console/app/app.py:66-93`
- `api/controllers/console/app/app.py:459-532`
- `api/services/app_service.py:56-100`

## Doc 7: Interview Walkthrough

Concise explanation:

"Dify is a full-stack LLM application platform. The frontend is a Next.js/React app with React Query, typed and manual API clients, Zustand for workflow state, and route-group-based layouts. The backend is Flask with RESTX controllers, Pydantic validation, service-layer business logic, SQLAlchemy models, Celery/Redis for async work, and Dockerized middleware. A typical request goes from a React component through a service hook, into a Flask controller with auth and tenant checks, then into services and database models."

Deeper follow-ups:

- Routing: backend blueprints are registered in `api/extensions/ext_blueprints.py:25-100`; frontend routes live under `web/app`.
- Auth: `login_required` checks authentication and CSRF in `api/libs/login.py:85-100`.
- Tenant isolation: `get_app_model` scopes app lookup by tenant in `api/controllers/console/app/wraps.py:13-18`.
- Workflow complexity: run orchestration is in `web/app/components/workflow-app/hooks/use-workflow-run.ts:191-441` and backend route in `api/controllers/console/app/workflow.py:697-733`.
- Trade-off: model properties like `App.site` improve ergonomics but can hide DB cost.

