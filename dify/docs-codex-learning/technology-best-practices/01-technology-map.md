# Technology Map

## pnpm Monorepo Tooling

**Role in This Project:** Coordinates root scripts, frontend workspace commands, package manager version, and development workflows.

**Where It Appears:** `package.json`, `web/package.json`, `e2e/package.json`, `pnpm-workspace.yaml`.

**Why It Matters:** Many commands must run from the correct working directory. Root `pnpm type-check` delegates into `web`, while frontend tests run from `web/`.

**What to Study First:** `package.json`, `web/package.json`, `e2e/package.json`.

**Related Technologies:** Next.js, Vitest, Cucumber, linting, type checking.

**Learning Priority:** High. Misrunning commands is one of the fastest ways to lose time in a monorepo.

## Next.js App Router

**Role in This Project:** Provides the frontend application shell, routing conventions, metadata, layout composition, and server/client component boundaries.

**Where It Appears:** `web/app/layout.tsx:29-91`, route folders under `web/app/`.

**Why It Matters:** The root layout wires providers that every feature relies on: Jotai, theme, React Query, i18n, toast, tooltip, and update checking.

**What to Study First:** `web/app/layout.tsx:29-91`, `web/context/app-context-provider.tsx:30-159`, `web/context/query-client.tsx:1-28`.

**Related Technologies:** React, React Query, Jotai, i18n, Tailwind, Next.js metadata.

**Learning Priority:** High. If you misunderstand the root layout, feature behavior feels magical.

## React

**Role in This Project:** Builds Dify's web UI, including dataset screens, workflow editor, settings pages, and reusable components.

**Where It Appears:** `web/app/components/develop/tag.tsx:4-60`, `web/app/components/datasets/list/index.tsx:25-108`, `web/app/components/workflow-app/index.tsx:40-155`.

**Why It Matters:** Most product changes in the web app are component changes plus hook/service changes.

**What to Study First:** `web/app/components/develop/tag.tsx:4-60`, `web/app/components/datasets/list/datasets.tsx:11-70`, `web/app/components/datasets/list/index.tsx:25-108`.

**Related Technologies:** TypeScript, React Query, Zustand, Jotai, Tailwind, i18n.

**Learning Priority:** High.

## TypeScript

**Role in This Project:** Describes component props, hook inputs, service payloads, contracts, and UI state.

**Where It Appears:** Nearly all `web/**/*.ts` and `web/**/*.tsx` files.

**Why It Matters:** TypeScript is the frontend's first review layer. It catches bad payloads, invalid props, and unsafe assumptions before runtime.

**What to Study First:** `web/app/components/datasets/list/datasets.tsx:11-38`, `web/contract/console/apps.ts:6-47`, `web/service/base.ts:41-132`, `web/service/fetch.ts:21-24`.

**Related Technologies:** React, oRPC, zod, generated contracts, service wrappers.

**Learning Priority:** High. Also watch for places where `any` weakens safety.

## React Query

**Role in This Project:** Manages server state, cache keys, invalidation, loading states, infinite lists, and mutation side effects.

**Where It Appears:** `web/context/query-client.tsx:1-28`, `web/service/knowledge/use-dataset.ts:28-104`, `web/service/client.ts:87-137`.

**Why It Matters:** React Query is how frontend screens stay synchronized with backend data without manually writing global server-state stores.

**What to Study First:** `web/service/knowledge/use-dataset.ts:28-104`, `web/context/query-client.tsx:1-28`.

**Related Technologies:** ky wrapper, oRPC client, REST endpoints, component rendering.

**Learning Priority:** High.

## Zustand

**Role in This Project:** Holds complex client-side workflow editor state.

**Where It Appears:** `web/app/components/workflow-app/store/workflow/workflow-slice.ts:3-35`, workflow editor folders.

**Why It Matters:** Some UI state is not server state. Workflow graph editing needs local mutation, selection, and runtime state that would be awkward in React Query.

**What to Study First:** `web/app/components/workflow-app/index.tsx:40-155`, `web/app/components/workflow-app/store/workflow/workflow-slice.ts:3-35`, `web/app/components/workflow-app/hooks/use-workflow-run.ts:99-190`.

**Related Technologies:** React Flow, hooks, SSE, backend workflow routes.

**Learning Priority:** High for workflow features, medium elsewhere.

## Jotai

**Role in This Project:** Provides atom-based state where small shared state units are useful.

**Where It Appears:** Provider wiring in `web/app/layout.tsx:61-84`.

**Why It Matters:** It is part of the app shell. Before adding new global state, study whether existing Jotai, Zustand, or React Query patterns better match the problem.

**What to Study First:** `web/app/layout.tsx:61-84`, then search for atom definitions in `web/`.

**Related Technologies:** React, Zustand, React Query.

**Learning Priority:** Medium.

## ky HTTP Wrapper

**Role in This Project:** Normalizes frontend HTTP calls, auth headers, CSRF handling, body serialization, response parsing, and error behavior.

**Where It Appears:** `web/service/fetch.ts:103-277`, `web/service/base.ts:181-187`.

**Why It Matters:** If a request fails, this wrapper is often where headers, tokens, body formatting, or error normalization should be inspected.

**What to Study First:** `web/service/fetch.ts:137-249`, `web/service/base.ts:181-187`.

**Related Technologies:** React Query, oRPC client, REST APIs, authentication.

**Learning Priority:** High.

## oRPC And OpenAPI Contracts

**Role in This Project:** Provides typed API contracts and clients for some frontend/backend interactions.

**Where It Appears:** `web/contract/router.ts:56-120`, `web/contract/console/apps.ts:6-47`, `web/service/client.ts:68-90`.

**Why It Matters:** Typed contracts reduce drift between frontend calls and backend API shape, but the repo currently has mixed manual/generated contract coverage.

**What to Study First:** `web/contract/console/apps.ts:6-47`, `web/service/client.ts:68-90`, `web/contract/router.ts:65-69`.

**Related Technologies:** TypeScript, zod, OpenAPI, backend Flask routes.

**Learning Priority:** Medium-high.

## Flask

**Role in This Project:** Runs the backend API application.

**Where It Appears:** `api/app.py:32-62`, `api/app_factory.py:49-138`, `api/extensions/ext_blueprints.py:25-100`.

**Why It Matters:** The backend's request lifecycle, extension setup, middleware hooks, and route registration all flow through Flask.

**What to Study First:** `api/app.py:32-62`, `api/app_factory.py:49-138`, `api/extensions/ext_blueprints.py:25-100`.

**Related Technologies:** Flask-RESTX, Pydantic, SQLAlchemy, Celery, Redis.

**Learning Priority:** High.

## Flask-RESTX

**Role in This Project:** Organizes REST resources, request parsing, route classes, and API namespaces.

**Where It Appears:** `api/controllers/console/app/app.py:459-561`, `api/controllers/console/datasets/datasets.py:289-450`.

**Why It Matters:** This is the main backend route/controller style.

**What to Study First:** `api/controllers/console/app/app.py:459-561`, `api/controllers/console/datasets/datasets.py:289-450`.

**Related Technologies:** Pydantic validation, service layer, decorators, response marshalling.

**Learning Priority:** High.

## Pydantic

**Role in This Project:** Validates request payloads and query parameters at backend API boundaries.

**Where It Appears:** `api/controllers/console/app/app.py:66-131`, `api/controllers/console/datasets/datasets.py:130-207`.

**Why It Matters:** Pydantic is where raw client input becomes trustworthy application input.

**What to Study First:** `api/controllers/console/app/app.py:66-93`, `api/controllers/console/datasets/datasets.py:130-149`.

**Related Technologies:** Flask-RESTX, services, SQLAlchemy models.

**Learning Priority:** High.

## SQLAlchemy

**Role in This Project:** Defines database-backed domain models and persistence relationships.

**Where It Appears:** `api/models/base.py:14-104`, `api/models/model.py:392-490`, `api/models/dataset.py:166-220`.

**Why It Matters:** Backend features eventually depend on model fields, relationships, constraints, and query behavior.

**What to Study First:** `api/models/base.py:14-104`, `api/models/model.py:392-490`, `api/models/dataset.py:166-220`.

**Related Technologies:** Flask services, migrations, PostgreSQL, service layer.

**Learning Priority:** High.

## Celery And Redis

**Role in This Project:** Support asynchronous work and task execution.

**Where It Appears:** Backend configuration and task tests such as `api/tests/unit_tests/tasks/test_workflow_execute_task.py:73-160`.

**Why It Matters:** Workflow execution and long-running jobs should not block HTTP request/response cycles.

**What to Study First:** `api/tests/unit_tests/tasks/test_workflow_execute_task.py:73-160`, then search task modules under `api/tasks`.

**Related Technologies:** Flask app context, Redis broker, workflow service.

**Learning Priority:** Medium-high.

## Vitest And React Testing Library

**Role in This Project:** Test frontend components and hooks.

**Where It Appears:** `web/app/components/develop/__tests__/tag.spec.tsx:1-140`, `web/app/components/workflow-app/hooks/__tests__/use-workflow-run.spec.ts:1-180`, `web/docs/test.md`.

**Why It Matters:** Tests express expected behavior and protect refactors.

**What to Study First:** `web/app/components/develop/__tests__/tag.spec.tsx:1-140`.

**Related Technologies:** React, hooks, mocks, test setup.

**Learning Priority:** High.

## pytest

**Role in This Project:** Tests backend units and task behavior.

**Where It Appears:** `api/tests/unit_tests/tasks/test_workflow_execute_task.py:73-160`, Makefile backend test commands.

**Why It Matters:** Backend refactors need tests around business logic and async behavior.

**What to Study First:** `api/tests/unit_tests/tasks/test_workflow_execute_task.py:73-160`.

**Related Technologies:** Flask app context, Celery tasks, services.

**Learning Priority:** High.

## Cucumber And Playwright

**Role in This Project:** Provide browser-level end-to-end testing.

**Where It Appears:** `e2e/package.json`.

**Why It Matters:** E2E tests validate user journeys across frontend and backend boundaries.

**What to Study First:** `e2e/package.json`, then feature files under `e2e/`.

**Related Technologies:** Next.js app, backend API, browser automation.

**Learning Priority:** Medium.

## Technologies Not Present As Primary Patterns

GraphQL, Express, NestJS, Prisma, TypeORM, Redux, and tRPC were not identified as primary stack choices in the inspected code paths. The equivalent learning areas are:

- GraphQL or tRPC style contracts: study oRPC and `web/contract/router.ts:56-120`.
- Express/NestJS controllers: study Flask-RESTX resources in `api/controllers/console/app/app.py:459-561`.
- Prisma/TypeORM persistence: study SQLAlchemy models in `api/models/model.py:392-490`.
- Redux global state: study Zustand and Jotai provider patterns.

