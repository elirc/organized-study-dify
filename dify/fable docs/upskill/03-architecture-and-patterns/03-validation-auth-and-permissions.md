# Validation, Auth, and Permissions

## The validation onion (every layer, mapped)

| Layer | Mechanism | Example |
| --- | --- | --- |
| Browser | TS types + form components | typed bodies in `chat/hooks.ts#L685-L692` (types don't *enforce* anything at runtime — say this out loud) |
| HTTP edge | global `before_request` | enterprise license gate `app_factory.py#L59-L96` |
| Route decorators | authn/authz/setup | `completion.py#L155-L159` stack |
| Payload | Pydantic models + validators | `ChatMessagePayload` UUID validation `completion.py#L55-L68` |
| Service | business rules, quotas, plan limits | batch/plan/vector-space checks `dataset_service.py#L1982-L2006` |
| Task | re-validation after the queue | same checks repeated `document_indexing_task.py#L64-L91` |
| DB | constraints, enums, NOT NULL | `apps` table defs `model.py#L392-L424` |

Rule of thumb worth quoting: **validate at every trust boundary, not every function**. The queue is a trust boundary (time passes, state changes) — hence the duplicated checks in the task. Between service and core in-process, re-validation would be noise.

## Authentication — three front doors

| Surface | Mechanism | Code |
| --- | --- | --- |
| Console (`/console/api`) | JWT access token + Redis refresh token + cookies + **CSRF header** | issue: `account_service.py#L200-L212`; enforce: `libs/login.py#L86-L100` (CSRF at L99); refresh storage `#L150-L159` |
| Service API (`/v1`) | Bearer app token (`ApiToken`), cached with single-flight | `service_api/wraps.py#L55-L120` |
| Web app (`/api`, shared pages) | Short-lived "passport" JWT per share code | `libs/passport.py`; headers set in `web/service/base.ts#L502-L506` |

All JWTs are HS256 with the single `SECRET_KEY` (`passport.py#L8-L13`). Consequence to articulate: key rotation logs everyone out and invalidates all passports; a leaked key forges any identity. (Mitigations to name if asked: key versioning/kid headers, asymmetric keys for cross-service verification.)

CSRF: needed because console auth rides on cookies. Token delivered as a cookie, echoed as a header (`web/service/fetch.ts#L184`), verified server-side per request (`libs/login.py#L99`) — the standard double-submit pattern. The service API doesn't need CSRF (no cookies; the bearer token *is* the intent).

## Authorization — roles and resource scoping

- **Roles per tenant**: `TenantAccountRole` (`account.py#L19`) — owner/admin/editor/normal/dataset_operator; helpers like `has_edit_permission` (`#L210-L219`).
- **Decorators**: `edit_permission_required` (`console/wraps.py#L306-L321`), `is_admin_or_owner_required` (`#L324-L337`). Note chat *debugging* requires edit permission (`completion.py#L159`) — authz applied to a "read-ish" endpoint because it consumes paid model quota. Authorization is about resources, not verbs.
- **Resource scoping (the IDOR defense)**: `get_app_model` loads the app *by id and tenant*; the repo rule is "Always scope queries by `tenant_id`" (`api/AGENTS.md`, SQLAlchemy section). Every place someone writes `select(X).where(X.id == id)` **without** `tenant_id` is a candidate cross-tenant read — this is the #1 thing to check in review here.
- **Dataset-level permissions**: beyond roles, datasets have per-user permissions (`DatasetPermission`, `dataset.py#L1355`; `DatasetPermissionEnum` L160 — only_me / all_team / partial).

## What a junior misses vs what a senior checks

Junior misses: that `current_user` can be an `EndUser` (not just `Account`) — controllers assert the type (`completion.py#L172-L173`); that authn ≠ authz; that Pydantic runs *after* the auth decorators (don't leak resource existence to unauthenticated callers).

Senior checks: the service-API **owner fallback** (`service_api/wraps.py#L104-L120`) — token-only calls execute with the *tenant owner's* account identity; any new service-API endpoint inherits owner-level power. Also: the end-user `user` field is caller-chosen (`#L82-L99`) — an app token owns its end-user namespace; two callers sharing a token share users. Both are design choices with real blast radius; knowing where they live is senior-level repo knowledge.

## Fake-code contrast

```python
# Illustrative fake code: not from this repo
# BAD — id-only lookup, no tenant scope (IDOR)
doc = session.scalar(select(Document).where(Document.id == doc_id))

# BETTER — scope to the caller's tenant every time
doc = session.scalar(select(Document).where(
    Document.id == doc_id,
    Document.tenant_id == current_tenant_id,
))
```

The real pattern, with `dataset_id` as the scope: `document_indexing_task.py#L82-L84` (`Document.id == document_id, Document.dataset_id == dataset_id`).

## Drills

1. Trace a request to `GET /v1/...` with a *revoked* API token. Where exactly does it die, and how stale can the token cache be? (Read `services/api_token_service.py`.) Strong = you can propose a cache-invalidation-on-revoke design.
2. List every check between TCP accept and the first line of `ChatMessageApi.post` body, in order, with file:line. (Answer sketch: license before_request → setup → login+CSRF → account init → app load+mode → edit permission → Pydantic.)
3. Grep `api/services/` for one query missing tenant scoping that handles a user-supplied id. If you find a real one, write it up per [09-reference/risk-register.md](../09-reference/risk-register.md) format ("investigate," not "bug").

## Interview angle

- "What's IDOR and how do you prevent it?" → tenant-scoped queries as a codified rule + the decorator that loads resources scoped. (→ 08/03)
- "JWT logout is hard — how do you do it?" → refresh tokens in Redis, deletable (`account_service.py#L157-L159`); short access-token TTL bounds the exposure. (→ 08/03)
- "Where do you put validation?" → the trust-boundary rule with the queue-revalidation example. (→ 08/03, 08/05)
