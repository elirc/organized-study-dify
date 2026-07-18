# Senior Engineer Guide

## Architecture Scores

| Area | Score | Evidence | Notes |
|---|---:|---|---|
| Scalability | 4 | `api/app_factory.py:177-219`, `docker/docker-compose.yaml`, Celery/Redis deps in `api/pyproject.toml:10-22` | Mature split between web, API, workers, Redis, DB, storage, providers. |
| TypeScript discipline | 3 | `web/contract/console/apps.ts:6-23`, `web/app/components/workflow-app/store/workflow/workflow-slice.ts:10-11` | Strong contracts in places, but `any` remains in core UI state and fetch options. |
| Separation of concerns | 4 | `api/controllers/console/app/app.py:469-561`, `api/services/app_service.py:56-203` | Controllers mostly validate/delegate; services own persistence and side effects. |
| Testability | 4 | `web/docs/test.md:25-52`, `web/app/components/develop/__tests__/tag.spec.tsx:1-140`, `api/tests/unit_tests/tasks/test_workflow_execute_task.py:73-160` | Good unit patterns; integration tests are CI/local-container dependent. |
| Maintainability | 3 | `api/controllers/console/app/workflow.py`, `web/app/components/workflow-app/hooks/use-workflow-run.ts:191-441` | Complex areas need careful decomposition and contract tests. |
| Security posture | 4 | `api/libs/login.py:85-100`, `api/controllers/console/app/wraps.py:13-18`, `web/service/fetch.ts:181-185` | CSRF, tenant scoping, auth decorators visible. Must preserve tenant filters. |
| Performance | 3 | `api/models/model.py:441-468`, `api/models/dataset.py:213-220` | Some convenient model properties can create repeated DB reads. |
| Developer experience | 4 | `Makefile:17-46`, `web/docs/test.md`, `web/docs/lint.md` | Setup and quality paths exist; stack is still large and partially hybrid. |

## Performance Audit

Finding: hidden DB queries in model properties.

```python
# api/models/model.py:441-448
@property
def site(self) -> Site | None:
    return db.session.scalar(select(Site).where(Site.app_id == self.id))

@property
def app_model_config(self) -> AppModelConfig | None:
    if self.app_model_config_id:
        return db.session.scalar(select(AppModelConfig).where(AppModelConfig.id == self.app_model_config_id))
```

Why it matters: calling this inside a list serialization can turn one query into many. A corrected pattern for list endpoints would batch-load related rows in the service/controller and attach response data explicitly.

Suggested snippet for future work, not added:

```python
site_rows = db.session.scalars(select(Site).where(Site.app_id.in_(app_ids))).all()
sites_by_app_id = {site.app_id: site for site in site_rows}
```

Finding: dataset counts are property queries.

- `api/models/dataset.py:213-220` computes document totals from properties. In list views, prefer aggregate queries.

## Security Audit

Strong pattern: auth and CSRF are centralized.

```python
# api/libs/login.py:85-100
if user is None or not user.is_authenticated:
    return _get_login_manager().unauthorized()
g._login_user = user
check_csrf_token(request, user.id)
return current_app.ensure_sync(func)(*args, **kwargs)
```

Strong pattern: tenant-scoped app loading.

```python
# api/controllers/console/app/wraps.py:13-18
app_model = db.session.scalar(
    select(App).where(App.id == app_id, App.tenant_id == current_tenant_id, App.status == "normal").limit(1)
)
```

Risk to watch: any new endpoint that loads by id without tenant or permission checks. Suggested review rule: no backend resource lookup by primary key alone unless it is explicitly global and documented.

Frontend security boundary:

- CSRF header is added in `web/service/fetch.ts:181-185`.
- Public/share routes add app code/passport headers in `web/service/fetch.ts:103-114`.

## TypeScript Discipline Review

Good:

- API contracts in `web/contract/console/apps.ts:6-23`.
- Context value typing in `web/context/app-context.ts:7-73`.
- Workflow callback types in `web/service/base.ts:41-132`.

Weak:

- `FetchOptionType` allows `params?: Record<string, any>` and `body?: BodyInit | Record<string, any> | null` in `web/service/fetch.ts:21-24`.
- Workflow slice uses `Record<string, any>` for node configs in `web/app/components/workflow-app/store/workflow/workflow-slice.ts:10-11`.

Senior move: replace `any` only when the real domain shape is known. Otherwise, use `unknown` plus narrow validators at the boundary.

## Custom Abstractions Inventory

- Flask app factory/extension system: `api/app_factory.py:127-219`.
- Blueprint CORS registration: `api/extensions/ext_blueprints.py:13-100`.
- oRPC client bridge: `web/service/client.ts:49-90`.
- Fetch/request wrappers: `web/service/fetch.ts:137-249`, `web/service/base.ts:181-220`.
- App context provider: `web/context/app-context-provider.tsx:30-159`.
- Workflow Zustand slices: `web/app/components/workflow-app/store/workflow/workflow-slice.ts:1-35`.
- Workflow run callback factories: `web/app/components/workflow-app/hooks/use-workflow-run.ts:378-441`.

## Testing Assessment

Strengths:

- Frontend testing guide is explicit about behavior-first testing in `web/docs/test.md:40-52`.
- Simple component tests are readable and focused in `web/app/components/develop/__tests__/tag.spec.tsx:1-140`.
- Complex hook tests isolate dependencies with hoisted mocks in `web/app/components/workflow-app/hooks/__tests__/use-workflow-run.spec.ts:32-180`.
- Backend task tests use monkeypatch/fakes to avoid full infrastructure in `api/tests/unit_tests/tasks/test_workflow_execute_task.py:73-160`.

Gap: app list controller/service behavior would benefit from focused tests around tag filtering, tenant scoping, and invalid UUID handling.

Example missing test, documentation only:

```python
def test_app_list_rejects_invalid_tag_id(client, logged_in_account):
    response = client.get("/console/api/apps?tag_ids[]=not-a-uuid")
    assert response.status_code == 400
    assert "Invalid UUID format" in response.get_data(as_text=True)
```

## Bug Injection Exercise

1. Symptom: dataset search returns old results after clearing a filter. Test: change `tag_ids`, clear them, assert query key causes refetch.
2. Symptom: a user from workspace A can open an app from workspace B. Test: create two tenants, call app detail with other tenant app id, expect 404.
3. Symptom: workflow debug run continues after stop. Test: start run, call stop with no task id, assert all abort controllers are invoked.
4. Symptom: app list crashes when tag IDs arrive as `tag_ids[0]`. Test: use bracketed query params and assert normalized UUID list.
5. Symptom: public webapp login loops. Test: simulate unauthorized bootstrap endpoint and ensure exempt endpoints do not trigger forced logout.

## Git History Learning Exercise

Recent commits from `git log --oneline -n 12`:

- `9d0906c684 chore: improve swagger markdown optional fields typing` implies API schema/doc typing is actively maintained.
- `41b6f894c0 fix: fetch memory of LLM node may cause out of flask context` implies workflow execution crosses request/background context boundaries.
- `27b084c4d4 fix: reduce db roundtrips of message update` confirms performance work around DB query count matters.
- `a252fbddfa feat: initialize user timezone and language from browser` shows frontend user environment feeds backend/profile behavior.
- `b41338cd08 chore(layout): reintroduce AmplitudeProvider in common layouts` shows analytics/provider placement in layout is architecturally significant.

## If I Owned This Codebase

| Item | Effort | Impact | Risk | Why It Matters |
|---|---|---:|---|---|
| Batch-load related App/Site/config data in list paths | M | High | Medium | Reduces hidden query costs. |
| Continue replacing `any` in workflow store slices | M | Medium | Medium | Improves refactor safety in complex UI. |
| Document contract-generation adoption status | S | Medium | Low | Helps devs know manual vs generated API paths. |
| Add controller/service tests for app list filters | S | High | Low | Protects tenant and validation behavior. |
| Split workflow run hook by orchestration phases | L | High | High | Easier tests and fewer side-effect regressions. |
| Add endpoint checklist for tenant scoping | S | High | Low | Prevents security regressions. |

