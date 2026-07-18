# Senior Build Projects

Six projects, 2 days–4 weeks. Each includes the full senior checklist: problem, value, design decisions, likely files, migration, tests, security, performance, rollout/rollback, open questions, stretch. These are realistic — shaped like things maintainers might accept (check for existing issues/PRs upstream before investing).

---

## S1: Transactional outbox for task dispatch (2-3 weeks)
**Problem**: commit-then-enqueue (`dataset_service.py#L2269-L2277`) leaves a crash window where rows exist but no task runs.
**Value**: eliminates a whole class of "stuck forever" states across *every* dispatch site.
**Design decisions**: outbox table schema (id, task_name, payload_json, status, created_at, claimed_by); poller as Celery beat vs dedicated loop; at-least-once + idempotent tasks (already the house rule); per-queue routing preserved.
**Likely files**: new `models/outbox.py`, migration, `services/` dispatch helper replacing direct `.delay()` at chosen sites, poller in `schedule/`, tests under `unit_tests/services` + `unit_tests/tasks`.
**Migration plan**: introduce helper; convert *one* dispatch site (document indexing) behind config flag; measure; convert others incrementally.
**Test plan**: crash-window simulation (commit, no enqueue, poller recovers); duplicate-delivery idempotency; poison-payload quarantine.
**Security**: payloads may contain tenant data — same protections as task args today.
**Performance**: one extra insert per dispatch; poller batch size; index on (status, created_at).
**Rollout/rollback**: config flag per site; rollback = flip flag, drain outbox.
**Open questions**: is the incident rate worth the moving part? (Gather stuck-doc stats first — engineering is calibrated paranoia.)
**Stretch**: dead-letter UI in console.
**Interview story potential**: end-to-end reliability engineering with staged rollout — a flagship system-design story.

## S2: Indexing operations dashboard (1-2 weeks)
**Problem**: indexing health is only visible per-document; operators lack fleet view (stuck counts, queue depth, per-tenant backlog).
**Design**: console admin page reading aggregates (status counts by age bucket, Redis queue depths via `TenantIsolatedTaskQueue`); no new writes.
**Likely files**: new console controller + service (read-only), knowledge admin UI page, charts via existing `echarts` catalog entry.
**Tests**: aggregate queries tenant-scoped for non-admin surfaces; snapshot the SQL for index review.
**Perf**: aggregates on big tables need indexes or periodic materialization — decide with EXPLAIN.
**Rollout**: pure addition; feature-flag the route.
**Interview story**: "I built the observability that made a background system operable."

## S3: End-to-end typed contract: OpenAPI → generated client adoption for the console app surface (2-4 weeks)
**Problem**: hand-written frontend types drift from backend responses (02/03); migration is half-done (dual key systems).
**Design**: backend response models registered per `controllers/API_SCHEMA_GUIDE.md`; generate with `@hey-api/openapi-ts`; adopt per-resource with key factories; delete legacy fetchers per resource.
**Likely files**: `api/openapi/`, `web/service/client.ts`, per-resource hooks, codegen script in `web/scripts/`.
**Migration**: resource-by-resource (M10 is the pilot); track a burndown of legacy `service/*.ts` exports (knip helps find dead ones).
**Tests**: contract test comparing generated types against a recorded response per resource; stale-cache regression per converted mutation.
**Rollback**: per-resource, trivial (keep legacy file until deletion PR).
**Interview story**: leading an incremental migration with drift-prevention — extremely strong mid→senior evidence.

## S4: Per-app webhook on workflow completion (2-3 weeks)
**Problem**: external systems must poll for workflow results; a signed webhook on run completion closes the loop.
**Design decisions**: delivery via Celery `mail`-like dedicated queue; HMAC signature (key per app, stored encrypted like provider creds); retries with backoff + max attempts + dead-letter status; **SSRF**: destination fetched via `ssrf_proxy` helper (house rule); event payload versioned (`v1` envelope).
**Likely files**: new model (webhook endpoints), migration, service, task in `tasks/`, settings UI under app configuration, service-API management endpoints.
**Tests**: signature verification vector tests; retry schedule; SSRF-proxy enforced (test that raw egress is refused); tenant isolation on management endpoints.
**Security**: replay protection (timestamp + tolerance), secret rotation with dual-accept window.
**Rollout**: per-app opt-in; global kill switch config.
**Interview story**: outbound webhooks done right — hits security, reliability, and API design in one narrative.

## S5: Conversation retention policies (2-3 weeks)
**Problem**: conversations/messages grow unbounded; tenants (and GDPR) want retention windows.
**Design**: per-tenant retention setting; scheduled deletion task batching by id ranges; interaction with annotations/feedback (`MessageFeedback` L1782, `MessageAnnotation` L1861 in `model.py`) — cascade or preserve?; existing retention code under `services/retention/` — **extend, don't duplicate** (read it first).
**Migration**: setting default = keep-forever (no behavior change on ship).
**Tests**: boundary (exactly N days), tenant isolation of deletion, dry-run mode (T15 pattern).
**Perf**: batched deletes with sleep intervals; index on (tenant, created_at).
**Rollback**: deletion is irreversible — hence dry-run default-on for the first release and per-tenant opt-in.
**Interview story**: data-lifecycle engineering with irreversibility handling — rare and valuable.

## S6: Workflow node execution cache (3-4 weeks, research-flavored)
**Problem**: deterministic nodes (code, template, HTTP GET with same inputs) re-run on every draft-debug iteration, costing time and tokens.
**Design decisions**: cache key = node id + normalized inputs hash + node version; storage in Redis with TTL; **only** for `InvokeFrom.DEBUGGER` runs (never production semantics); explicit per-node opt-out; cache-hit surfaced in the run UI (honesty!).
**Likely files**: a new engine layer (Pattern 12 — `workflow_entry.py#L199-L220` shows the layering seam), debug-run service, run-panel UI badge.
**Tests**: key stability across dict ordering; miss on input change; never active for published runs.
**Open questions**: which node types are safely deterministic? (LLM nodes are not, by default.)
**Interview story**: performance feature with correctness boundaries — shows restraint (debugger-only) as a design value.

---

**Meta-advice**: for a portfolio, S1 or S3 done *partially but publicly* (design doc + first PR) beats S6 done silently. The design doc is half the value — write it per [07/02](../07-career-and-collaboration/02-writing-prs-and-rfcs.md).
