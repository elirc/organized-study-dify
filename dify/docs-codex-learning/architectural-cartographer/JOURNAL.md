# Architectural Cartographer Journal

## First-Pass Mental Model

Dify is a full-stack monorepo for building and operating LLM applications. The backend is a Flask application with Flask-RESTX controllers, Pydantic request models, service classes, SQLAlchemy models, Celery tasks, Redis, storage, and many provider plugins. The frontend is a Next.js/React/TypeScript application using App Router, React Query, oRPC/OpenAPI-style contracts, Zustand, Jotai, Tailwind, and i18n resources.

Evidence:

- Root package manager and monorepo scripts: `package.json:1-18`.
- Frontend dependencies and scripts: `web/package.json:25-43`, `web/package.json:65-153`, `web/package.json:157-230`.
- Backend dependencies: `api/pyproject.toml:1-47`, `api/pyproject.toml:114-181`.
- Backend boot: `api/app.py:32-62`, `api/app_factory.py:127-138`.
- Frontend root layout: `web/app/layout.tsx:29-91`.

## Important Discoveries

- Backend extension loading is centralized and ordered in `api/app_factory.py:177-219`. This is the backend's "power strip": database, Redis, storage, Celery, login, blueprints, OpenAPI, tracing, request logging.
- Blueprints are registered with different CORS/header policies in `api/extensions/ext_blueprints.py:25-100`. This is where console, web app, files, service API, inner API, MCP, and triggers become externally reachable.
- The frontend root layout wires global providers in one place: Jotai, theme, query client, i18n, toast, tooltip, route prefix, and dev tooling in `web/app/layout.tsx:61-84`.
- Dify has both legacy/manual request wrappers and typed oRPC contract clients. `web/service/client.ts:68-90` wires typed console requests through the existing `request` wrapper.
- App listing is a strong full-stack teaching anchor because it touches `web/contract/console/apps.ts:6-23`, `web/service/client.ts:87-137`, `api/controllers/console/app/app.py:459-532`, `api/services/app_service.py:56-100`, and `api/models/model.py:392-468`.
- Workflow execution is the most instructive complex path. The frontend run hook prepares canvas state and sends SSE via `web/app/components/workflow-app/hooks/use-workflow-run.ts:191-441`; the backend route accepts draft workflow runs in `api/controllers/console/app/workflow.py:697-733`.

## Why These Files Were Chosen

- `api/app.py`, `api/app_factory.py`, and `api/extensions/ext_blueprints.py` show how the backend exists at runtime.
- `web/app/layout.tsx` and `web/context/app-context-provider.tsx` show how the frontend gets global capabilities and workspace identity.
- `web/service/fetch.ts`, `web/service/base.ts`, `web/service/client.ts`, and `web/contract/router.ts` show request boundaries and type contracts.
- `api/controllers/console/app/app.py`, `api/services/app_service.py`, and `api/models/model.py` show a clean controller-service-model path.
- `web/app/components/workflow-app/index.tsx` and `use-workflow-run.ts` show realistic complexity: store coordination, graph data, replay, SSE, abort handling, and telemetry.
- Tests such as `web/app/components/develop/__tests__/tag.spec.tsx:1-140` and `api/tests/unit_tests/tasks/test_workflow_execute_task.py:73-160` show how behavior is verified.

## Where A Junior Might Get Confused

- The frontend uses route groups like `(commonLayout)` and `(shareLayout)`. These are Next.js organization tools, not URL path segments.
- There are two API styles in the frontend: typed oRPC contracts and direct `get/post` wrappers. Both are real.
- The backend has many controller folders. The key distinction is audience: console APIs, web app APIs, service APIs, inner APIs, trigger APIs.
- Models may perform database lookups through properties, for example `App.site` and `App.app_model_config` in `api/models/model.py:441-448`. That is convenient but can hide query costs.

## Where A Mid-Level Engineer Should Slow Down

- Query keys and invalidation. Example: dataset list keys in `web/service/knowledge/use-dataset.ts:28-104`.
- Permission boundaries. Example: `get_app_model` scopes apps by tenant in `api/controllers/console/app/wraps.py:13-18`.
- Request wrapper behavior. `web/service/fetch.ts:181-249` adds CSRF headers, prefixes URLs, handles marketplace/public APIs, and normalizes responses.
- Workflow run state. `web/app/components/workflow-app/hooks/use-workflow-run.ts:256-441` has several side effects that must stay coordinated.

## Where A Senior Engineer Should Be Skeptical

- Hidden database reads in model properties can become N+1 risks (`api/models/model.py:441-468`, `api/models/dataset.py:213-220`).
- Some frontend types still use `any` in core request options (`web/service/fetch.ts:21-24`) and Zustand slices (`web/app/components/workflow-app/store/workflow/workflow-slice.ts:10-11`).
- The app has both generated contracts and manual contracts. `web/contract/router.ts:65-69` explicitly says only enterprise generated contracts are wired into the console router so far.
- Workflow code is feature-rich and well-tested in places, but complexity is high enough that refactors need careful contract tests.

## Missing Or Weak Patterns

- No Prisma, TypeORM, Drizzle, Express, NestJS, Redux, GraphQL server, or tRPC app pattern was found as a main path. This is acceptable because the project is Flask/SQLAlchemy on the backend and Next/React on the frontend.
- Contract generation exists in `packages/contracts/generated`, but frontend console contracts are only partially wired. Impact: developers must know when to follow generated contracts versus manual service hooks.
- Setup is present in `Makefile:17-46`, but a first-time developer still needs judgment around Docker middleware, env files, API migrations, and which local services are actually required.

## How To Use Checkpoints

Answer checkpoints in writing. Strong answers should:

- cite a file and line range,
- identify the owner of each responsibility,
- describe data shape changes,
- name permission and tenant boundaries,
- predict one failure mode and where to debug first.

