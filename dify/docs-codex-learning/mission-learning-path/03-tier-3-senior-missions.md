# Tier 3 Senior Missions

### Mission 18: Reverse-Engineer The Architecture Decisions

**Tier:** Senior

**Time Estimate:** 60 minutes

**Goal:** Infer why the repo is structured the way it is.

**The Concept:** Architecture is fossilized decision-making. Read it like a history of constraints.

**Design Intent Before You Read the Code:** Dify uses a Flask backend with layered controllers/services/models, a Next frontend, worker infrastructure, provider plugins, and a mixed API contract strategy because the product spans UI, workflow execution, RAG, model providers, and public/private API surfaces.

**Find It In The Code:** `api/app_factory.py:177-219`, `api/pyproject.toml:10-47`, `web/contract/router.ts:65-69`.

```ts
// web/contract/router.ts:65-69
// Enterprise generated contract is wired in.
// Community API contracts are generated too, but not complete enough to consume directly yet.
```

**The Aha Moment:** Hybrid patterns often reveal migration in progress, not indecision.

**Socratic Checkpoint:** Why separate console/web/service APIs? Why use Celery and Redis? Why preserve manual API clients? Why use provider packages? Why is extension order explicit?

**How to Self-Grade:** Strong answers cite evidence and state uncertainty honestly.

**Connects To:** Mission 21 and Mission 24.

### Mission 19: Find The Bugs Before They Happen

**Tier:** Senior

**Time Estimate:** 60 minutes

**Goal:** Predict likely bugs from code structure.

**The Concept:** Senior debugging starts before runtime; you scan for pressure points.

**Design Intent Before You Read the Code:** Look for cache keys, hidden DB calls, tenant scoping, stale state, and side-effect ordering.

**Find It In The Code:** `web/service/knowledge/use-dataset.ts:83-104`, `api/models/model.py:441-468`, `web/app/components/workflow-app/hooks/use-workflow-run.ts:256-441`.

```ts
// web/service/knowledge/use-dataset.ts:86-95
queryKey: [...datasetListQueryKey, initialPage, tag_ids, limit, include_all, keyword]
// If a result-changing input is omitted here, React Query can reuse stale data.
```

**The Aha Moment:** Many production bugs are invisible contract mismatches.

**Socratic Checkpoint:** Which query key is easiest to break? Which model property can hide N+1 queries? Which workflow side effect must happen before SSE? Which tenant check protects app detail? Which error should be logged rather than swallowed?

**How to Self-Grade:** Strong answers name one frontend, one backend, and one workflow risk.

**Connects To:** Mission 20.

### Mission 20: The Bug Injection Challenge

**Tier:** Senior

**Time Estimate:** 50 minutes

**Goal:** Design tests for five realistic bugs without modifying production code.

**The Concept:** A good bug report is a failing scenario plus the first place to inspect.

**Design Intent Before You Read the Code:** Each bug should be described by user-visible symptom and test scenario, not implementation detail.

**Find It In The Code:** `web/app/components/datasets/list/index.tsx:35-52`, `api/controllers/console/app/wraps.py:13-18`, `api/controllers/console/app/workflow.py:735-759`.

```text
Bug scenario example:
Symptom: after clearing dataset search, old results remain.
Test: type keyword, wait for debounce, clear input, assert hook refetches with empty keyword.
First file: web/service/knowledge/use-dataset.ts:83-99.
```

**The Aha Moment:** You do not need to edit code to practice debugging judgment.

**Socratic Checkpoint:** What symptom would a tenant bug show? What test proves stop-run works? What stale cache bug is likely? What invalid UUID bug is likely? What setup/license loop is likely?

**How to Self-Grade:** Strong answers are testable, user-visible, and cite first inspection files.

**Connects To:** Mission 23.

### Mission 21: Performance X-Ray

**Tier:** Senior

**Time Estimate:** 60 minutes

**Goal:** Identify likely performance pressure points.

**The Concept:** Performance x-ray means looking through convenience APIs to see real work.

**Design Intent Before You Read the Code:** Model properties and list endpoints deserve scrutiny because they can multiply database queries.

**Find It In The Code:** `api/models/model.py:441-468`, `api/models/dataset.py:213-220`, `api/services/app_service.py:93-98`.

```python
# api/models/dataset.py:213-220
@property
def total_documents(self):
    return db.session.scalar(select(func.count(Document.id)).where(Document.dataset_id == self.id)) or 0
```

**The Aha Moment:** Convenient properties can become expensive when called in loops.

**Socratic Checkpoint:** Which properties query the DB? Which endpoint returns lists? What should be batch-loaded? What recent git commit hints at DB roundtrip work? What metric would you inspect in production?

**How to Self-Grade:** Strong answers propose batching or aggregation without rewriting unrelated architecture.

**Connects To:** Mission 18.

### Mission 22: The Security Audit

**Tier:** Senior

**Time Estimate:** 60 minutes

**Goal:** Audit auth, CSRF, tenant scoping, and public/share boundaries.

**The Concept:** Security posture is made of boring repeated correctness.

**Design Intent Before You Read the Code:** Central wrappers should enforce identity, CSRF, setup, tenant ownership, and permissions before service logic runs.

**Find It In The Code:** `api/libs/login.py:85-100`, `api/controllers/console/app/wraps.py:13-84`, `web/service/fetch.ts:181-185`, `web/service/fetch.ts:103-114`.

```python
# api/controllers/console/app/wraps.py:13-18
select(App).where(App.id == app_id, App.tenant_id == current_tenant_id, App.status == "normal")
```

**The Aha Moment:** Tenant scoping is a product security feature, not just a query filter.

**Socratic Checkpoint:** Where is CSRF checked? Where are share headers set? Which app lookup is tenant-scoped? What should a new delete endpoint verify? What is the danger of direct `db.session.get(App, id)`?

**How to Self-Grade:** Strong answers include one frontend and one backend security boundary.

**Connects To:** Mission 15.

### Mission 23: Write The Test That Doesn't Exist

**Tier:** Senior

**Time Estimate:** 75 minutes

**Goal:** Design a high-value missing test.

**The Concept:** Tests are executable memory for bugs you do not want to rediscover.

**Design Intent Before You Read the Code:** App list validation/tenant filtering is risky enough to deserve tests because it protects data boundaries.

**Find It In The Code:** `api/controllers/console/app/app.py:66-93`, `api/services/app_service.py:65-100`, `web/docs/test.md:40-52`.

```python
# Documentation-only test sketch
def test_app_list_invalid_tag_id_returns_400(client, logged_in_account):
    response = client.get("/console/api/apps?tag_ids[]=not-a-uuid")
    assert response.status_code == 400
```

**The Aha Moment:** The best missing tests protect contracts and boundaries, not implementation trivia.

**Socratic Checkpoint:** What behavior is risky? What setup is needed? What should be mocked? What should be real? What assertion proves the contract?

**How to Self-Grade:** Strong answers avoid brittle internals and verify observable behavior.

**Connects To:** Mission 20.

### Mission 24: The Git History Tells A Story

**Tier:** Senior

**Time Estimate:** 40 minutes

**Goal:** Use recent commits to infer active maintenance themes.

**The Concept:** Git history is the codebase's weather report.

**Design Intent Before You Read the Code:** Recent commits reveal what areas change, break, or receive investment.

**Find It In The Code:** Run `git log --oneline -n 12`.

```text
9d0906c684 chore: improve swagger markdown optional fields typing
41b6f894c0 fix: fetch memory of LLM node may cause out of flask context
27b084c4d4 fix: reduce db roundtrips of message update
a252fbddfa feat: initialize user timezone and language from browser
b41338cd08 chore(layout): reintroduce AmplitudeProvider in common layouts
```

**The Aha Moment:** History points you toward fragile or actively evolving areas.

**Socratic Checkpoint:** Which commit hints at context-boundary bugs? Which hints at performance pressure? Which hints at frontend layout/provider importance? Which hints at docs/schema investment? What would you inspect before touching workflow memory?

**How to Self-Grade:** Strong answers infer themes without overclaiming.

**Connects To:** Mission 18.

