# Reading Map

## One-Paragraph Mental Model

Dify is a monorepo where a Next.js frontend lets users create, configure, run, and observe LLM applications, while a Flask backend owns authentication, tenant-aware APIs, workflow execution, datasets/RAG, provider integrations, database persistence, queues, and operational concerns. The frontend bootstraps providers in `web/app/layout.tsx`, fetches data through React Query and request/contract layers, and calls Flask-RESTX controllers that validate payloads with Pydantic, delegate to services, persist through SQLAlchemy models, and sometimes stream workflow events back to the UI.

## Top 10 Files To Read In Order

| Order | File | Why Read It Now | Before Reading | Explain After Reading |
|---|---|---|---|---|
| 1 | `package.json:1-18` | Establish package manager, Node version, root scripts. | Monorepo basics. | Why `pnpm` and `vp` matter here. |
| 2 | `Makefile:17-46` | Reconstruct local setup. | Docker/env/migrations basics. | What `make dev-setup` prepares and what it does not start. |
| 3 | `api/app.py:32-62` | Backend runtime entry point. | Flask app factory idea. | Difference between db command app and normal app. |
| 4 | `api/app_factory.py:49-138`, `api/app_factory.py:177-219` | See request hooks and extension loading. | Flask extensions. | Why extension order matters. |
| 5 | `api/extensions/ext_blueprints.py:25-100` | See routing surfaces. | Flask blueprints/CORS. | Which audiences each API surface serves. |
| 6 | `web/app/layout.tsx:29-91` | Frontend provider root. | Next App Router layout. | Which global capabilities every page receives. |
| 7 | `web/context/app-context-provider.tsx:30-159` | Workspace/user context. | React context and React Query. | How role booleans are derived. |
| 8 | `web/service/client.ts:68-137` | Typed contract client and query utils. | HTTP client wrappers. | How typed console queries reuse the older request function. |
| 9 | `api/controllers/console/app/app.py:459-561` | Controller example. | REST controller responsibility. | How list/create app endpoints validate and delegate. |
| 10 | `api/services/app_service.py:56-203` | Service and persistence example. | SQLAlchemy sessions. | How tenant filters, model defaults, commits, and events fit together. |

## Three Important Data Flows

1. App list flow: `web/contract/console/apps.ts:6-23` defines `/apps`, `web/service/client.ts:87-90` exposes `consoleQuery`, `api/controllers/console/app/app.py:469-532` handles the request, `api/services/app_service.py:57-100` queries tenant-scoped apps, and `api/models/model.py:392-468` defines persisted app shape.
2. Dataset list flow: `web/app/components/datasets/list/index.tsx:25-108` manages filters, `web/app/components/datasets/list/datasets.tsx:26-68` calls `useDatasetList`, `web/service/knowledge/use-dataset.ts:83-99` builds the query, and `api/controllers/console/datasets/datasets.py:289-379` validates and returns datasets.
3. Workflow run flow: `web/app/components/workflow-app/index.tsx:40-155` initializes workflow state, `web/app/components/workflow-app/hooks/use-workflow-run.ts:191-441` runs SSE, `api/controllers/console/app/workflow.py:697-733` receives the draft run, and `services/app_generate_service.py` is the next business-logic handoff.

## Pre-Reading Checklist

- Can I name the frontend app folder?
- Can I name the backend app folder?
- Do I know the package manager?
- Do I know the backend web framework?
- Do I know the frontend router style?
- Do I know where env examples live?
- Do I know which files register backend routes?
- Do I know which frontend file wraps React Query?
- Do I know where SQLAlchemy models live?
- Do I know where tests live for frontend, backend, and e2e?

## Red Flags Checklist For Review

- A backend query touches shared data without tenant scoping.
- A controller contains business logic instead of validation/delegation.
- A frontend component fetches directly when a service hook pattern already exists.
- A query key omits a filter that changes returned data.
- A mutation does not invalidate or update relevant cached data.
- A route accepts unvalidated `dict[str, Any]` when a Pydantic model would fit.
- A model property hides repeated database calls in a loop.
- A workflow side effect is added without stop/abort/error behavior.
- A user-facing string is hardcoded instead of using `web/i18n/en-US/`.
- Tests mock a component so heavily that they no longer verify real behavior.

