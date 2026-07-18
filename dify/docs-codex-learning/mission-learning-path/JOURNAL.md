# Mission Design Journal

## Rationale

The missions are ordered from "find the heartbeat" to "own the system." Dify is large enough that a learner can get lost in feature folders, so the first missions build a mental map. The middle missions force end-to-end tracing through React Query, typed contracts, Flask controllers, Pydantic, services, SQLAlchemy, and workflow streaming. The senior missions train skepticism: performance, security, missing tests, and architecture evolution.

## Code Paths Chosen

- Boot and setup: `package.json:1-18`, `Makefile:17-46`, `api/app.py:32-62`, `api/app_factory.py:127-219`, `web/app/layout.tsx:61-84`.
- Frontend data: `web/service/client.ts:68-137`, `web/service/knowledge/use-dataset.ts:28-104`, `web/context/app-context-provider.tsx:30-159`.
- Backend route/service/model: `api/controllers/console/app/app.py:459-561`, `api/services/app_service.py:56-203`, `api/models/model.py:392-468`.
- Complex workflow: `web/app/components/workflow-app/index.tsx:40-155`, `web/app/components/workflow-app/hooks/use-workflow-run.ts:191-441`, `api/controllers/console/app/workflow.py:697-759`.
- Tests: `web/app/components/develop/__tests__/tag.spec.tsx:1-140`, `api/tests/unit_tests/tasks/test_workflow_execute_task.py:73-160`.

## Tier Skill Development

- Junior: identify entry points, name layers, read props, read Pydantic models, and follow basic routes.
- Mid-level: trace data across layers, understand state ownership, read side effects, review diffs, and reason about contracts.
- Senior: predict failure modes, review architecture, identify security/performance risks, write missing tests, and learn from history.

## What A Senior Would Do Differently

A senior reads for invariants, not just behavior. They ask: "Where is tenant isolation enforced? What validates this input? What happens when the cache is stale? What is the rollback path? What would make this slow at scale?"

## Adapted Gaps

This repo does not use Express, NestJS, Prisma, TypeORM, Redux, tRPC, or a primary GraphQL server. Missions therefore focus on Flask, SQLAlchemy, React Query, Zustand, Pydantic, oRPC contracts, and Dify-specific workflow/RAG domains.

