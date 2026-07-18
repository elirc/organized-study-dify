# User Story Journal

## Why These Stories Were Chosen

Dify's core domain is creating, organizing, configuring, running, and observing LLM apps and knowledge datasets. The stories therefore focus on real product actions: improving list filters, display signals, app metadata, dataset permissions, workflow debug behavior, and auditability.

## Domain Mapping

- App list and app metadata stories map to `api/controllers/console/app/app.py`, `api/services/app_service.py`, and frontend contracts in `web/contract/console/apps.ts`.
- Dataset stories map to `web/app/components/datasets/list`, `web/service/knowledge/use-dataset.ts`, and `api/controllers/console/datasets/datasets.py`.
- Workflow stories map to `web/app/components/workflow-app` and `api/controllers/console/app/workflow.py`.

## Safest Stories

Stories 1-3 are safest because they mostly touch UI and small validation boundaries. Story 1 can be done by studying `web/app/components/develop/tag.tsx:43-60` and its tests.

## Deeper Architecture Stories

Stories 8-10 require understanding frontend contracts, backend validation, tenant scoping, service logic, database models, and tests.

## Repeated Files And Patterns

- React Query hooks: `web/service/knowledge/use-dataset.ts:28-104`.
- oRPC contracts: `web/contract/console/apps.ts:6-35`.
- Backend Pydantic payloads: `api/controllers/console/app/app.py:66-175`.
- Service filters: `api/services/app_service.py:56-100`.
- Permission wrappers: `api/controllers/console/app/wraps.py:13-84`.

## What A Senior Would Watch

- Avoid missing query-key inputs.
- Keep tenant isolation mandatory.
- Do not put business logic in controllers.
- Add tests around validation, permissions, and cache invalidation.
- Avoid broad workflow refactors while implementing narrow behavior.

