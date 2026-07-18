# Reference Artifacts

### Mission 25: Write The Docs That Don't Exist

**Tier:** Senior

**Time Estimate:** 90 minutes

**Goal:** Turn your repo understanding into a short guide for the next engineer.

**The Concept:** Documentation is engineering empathy made durable.

**Design Intent Before You Read the Code:** Good docs cite files, explain why, identify boundaries, and warn about mistakes.

**Find It In The Code:** Use any mission anchor, especially `api/app_factory.py:177-219`, `web/service/client.ts:68-137`, and `api/controllers/console/app/app.py:459-561`.

```text
Doc skeleton:
- What this area owns
- Files to read first
- Normal data flow
- Permission and validation boundaries
- Common mistakes
- Test path
```

**The Aha Moment:** If you cannot document a flow, you do not fully own it yet.

**Socratic Checkpoint:** Who is the reader? What can they safely change? What should they not change casually? What line ranges prove your claims? What is the first test they should run?

**How to Self-Grade:** Strong docs are short, cited, and actionable.

**Connects To:** The user-story build path.

## Doc 1: Junior Onboarding Checklist

- Read root `package.json`, `Makefile`, `api/app.py`, `web/app/layout.tsx`.
- Identify frontend/backend/test folders.
- Trace `Tag` and its test.
- Trace one route: app list.
- Ask: what validates input, what checks auth, what writes data?

## Doc 2: Architecture Guide For New Engineers

- Start with boot: `api/app_factory.py:127-219`.
- Map route surfaces: `api/extensions/ext_blueprints.py:25-100`.
- Map frontend providers: `web/app/layout.tsx:61-84`.
- Map data clients: `web/service/fetch.ts:137-249`, `web/service/client.ts:68-137`.
- Trace one model-backed feature before editing.

## Doc 3: Code Review Checklist

- Does this change preserve tenant scoping?
- Are request inputs validated?
- Is business logic outside controllers?
- Are query keys complete?
- Are cache invalidations correct?
- Are user-facing strings in i18n?
- Are tests behavior-focused?

## Doc 4: Debugging Playbook

- Frontend stale data: inspect local state, query key, hook URL, backend route.
- Auth failure: inspect `api/libs/login.py:85-100` and browser redirects.
- Tenant leak: inspect resource loaders like `api/controllers/console/app/wraps.py:13-18`.
- Workflow run failure: inspect `use-workflow-run.ts:191-441`, backend route `workflow.py:697-733`, and worker logs.

## Doc 5: Change Playbook

1. Create branch.
2. Find similar feature.
3. Write failing test or manual verification notes.
4. Change frontend contract/hook/UI.
5. Change backend payload/controller/service/model if needed.
6. Run focused tests.
7. Review diff for unrelated changes.
8. Open PR with risk and verification.

## Doc 6: Senior Ownership Notes

- Monitor DB roundtrips in list endpoints.
- Keep tenant filters boring and consistent.
- Reduce `any` at meaningful boundaries.
- Keep workflow orchestration covered by tests before refactoring.
- Leave mature extension/blueprint boot order alone unless there is a clear reason.

## Doc 7: Interview Walkthrough

Practice answer:

"I studied Dify, a full-stack LLM app platform. The frontend is Next.js/React with React Query, Zustand, Jotai, i18n, and typed/manual API clients. The backend is Flask with RESTX controllers, Pydantic validation, service-layer logic, SQLAlchemy models, Celery/Redis, and provider plugins. I traced app listing from a typed frontend contract through a Flask route, Pydantic validation, tenant-scoped service filtering, SQLAlchemy persistence, and JSON response."

Trade-off answer:

"One strength is clear layering in controller-service-model flows. One risk is hidden DB work in model properties, which can create performance issues in list endpoints. I would add tests and batching before making large refactors."

