# Data Model and Persistence

## The entity map that matters

```
Tenant (workspace) ─┬─< TenantAccountJoin >─ Account
                    ├─< App ──┬── AppModelConfig (versioned config, classic apps)
                    │         ├── Workflow (versioned graph; one draft + published versions)
                    │         ├── Site (public share page)
                    │         └─< Conversation ─< Message (tree via parent linkage)
                    ├─< Dataset ─< Document ─< DocumentSegment ─< ChildChunk
                    │         └── DatasetProcessRule / DatasetPermission / DatasetMetadata
                    ├─< Provider (model credentials)
                    └─< ApiToken (service-api auth)
```

Anchors: `Tenant` `api/models/account.py#L240`; `Account` `#L87`; `App` `api/models/model.py#L392`; `Conversation` L1044 / `Message` L1393 (same file); `Dataset` `api/models/dataset.py#L166`, `Document` L487, `DocumentSegment` L833, `ChildChunk` L1047, `DatasetPermission` L1355.

Things to notice on the `App` table (`model.py#L392-L424`): almost every table indexes `tenant_id` (`app_tenant_id_idx`, L394) because *every* hot query filters by it; `mode` and `status` are text-backed enums; `workflow_id`/`app_model_config_id` are nullable pointers to the "current version" — a **versioning-by-pointer** pattern (new config row, then swing the pointer; old versions remain for history/rollback).

## Sessions and transactions — the three idioms

1. **Request-scoped `db.session`** (Flask-SQLAlchemy): used in controllers/services for simple work; committed explicitly. Example: document creation commits then dispatches (`dataset_service.py#L2269-L2277`).
2. **Explicit context-managed sessions** — house style for new code (`api/AGENTS.md`): `with Session(db.engine, expire_on_commit=False) as session:` … used where lifetimes must be controlled, e.g. `Account.current_tenant` setter (`account.py#L131-L144`) with a comment explaining `expire_on_commit=False` prevents `DetachedInstanceError` after close.
3. **Short transaction, long work outside**: the indexing task commits status `PARSING` in a `session.begin()` block, closes it, *then* runs the multi-minute pipeline sessionless (`document_indexing_task.py#L93-L120`). Never hold a transaction across network calls — this is the file to cite.

**Consistency expectations**: writes go through explicit commits; there's no distributed transaction between Postgres and the vector store — indexing is *eventually consistent* and the `indexing_status` column is the reconciliation record (ERROR rows carry the message, `indexing_runner.py#L58-L67`). When SQL and vector-store must both change (e.g., delete a document), order + idempotent cleanup tasks stand in for atomicity (`tasks/clean_document_task.py` et al.).

## Ordering invariant worth memorizing

Dispatch tasks **after** commit (`dataset_service.py#L2269` commit → `#L2271-L2277` delay). If you `.delay()` before commit, the worker can start before the row is visible → "document not found" heisenbugs. The safe general form is transactional-outbox; the pragmatic form used here is commit-then-enqueue (accepting a small "committed but enqueue failed" window). Know both names.

## Migrations: how to change the schema safely here

Mechanics: Alembic under `api/migrations/versions/` (171 files), run by `flask db upgrade`; a minimal app boots just db+migrate (`app_factory.py#L222-L230`).

Safe-change checklist (transferable):

1. **Additive first**: add nullable column / new table; deploy code that writes both; backfill; enforce NOT NULL later. Never rename in one step.
2. **Server defaults for hot tables**: note how columns here carry `server_default` (`model.py#L406-L424`) so old code inserting without the column still works mid-deploy.
3. **Indexes**: any new query filtered by more than `tenant_id` needs an index decision; big-table index creation is its own deployment risk (build concurrently in Postgres).
4. **Downgrade path**: Alembic `downgrade()` exists but assume roll-*forward* in practice; write reversible migrations anyway for review clarity.
5. **Data migrations separate from schema migrations**; long backfills belong in tasks/commands (`api/commands/` exists for this).

## Repository pattern — used *selectively*

`api/repositories/` and `core/repositories/` exist, and `api/AGENTS.md` says to introduce repositories "only for very large tables (e.g., workflow executions) or when alternative storage strategies are required." That restraint is itself a lesson: repositories everywhere is ceremony; repositories at scale/storage-swap points is engineering.

## Drills

1. Write the migration plan (steps, not code) to add `archived_at` to `apps` and exclude archived apps from all list queries. Rubric — Strong: nullable column + partial index consideration + which service methods change + how you find every list query (grep patterns) + rollout order.
2. Find the transaction boundary in `save_document_with_dataset_id` (`dataset_service.py#L1967-L2283`). How many commits happen? What happens if the process dies between commit and `.delay()`? Name the pattern that closes that gap (outbox).
3. `Conversation`/`Message` (`model.py#L1044`, `L1393`): sketch the query that loads a message *thread* (one root-to-leaf path) given the frontend's `parent_message_id` tree. Where would this get slow, and what index helps?

## Interview angle

- "Design the schema for a chat app with regenerable answers" → messages as a tree via parent id; show you know the sibling-branch consequence for pagination. (→ 08/03)
- "How do you change a schema with zero downtime?" → the additive checklist above, citing server defaults on `apps`. (→ 08/03)
- "SQL + a second store (search/vector) — how do you keep them in sync?" → status-column reconciliation + idempotent cleanup tasks, from Flow 3. (→ 08/04-system-design)
