# Architecture Critique

How a senior would assess this codebase after a week inside it. This file doubles as system-design interview material — see [08-interview-prep/04-system-design-from-this-repo.md](../08-interview-prep/04-system-design-from-this-repo.md).

## Strongest design choices

1. **Execution decoupled from delivery** (Flow 2): workflow runs in Celery; HTTP subscribes to a replayable event stream keyed by a pre-minted run id (`app_generate_service.py#L153-L185`). Buys: reconnect/reload resilience, worker scaling independent of web tier, pause/resume (human input) for free. Cost: Redis becomes availability-critical; two mental models for streaming vs blocking.
2. **Tenancy as a written invariant** with muscle behind it: `tenant_id` on every table + indexed (`model.py#L394`), scoping rule codified (`api/AGENTS.md`), tenant checks at the token boundary (`service_api/wraps.py#L73-L77`), per-tenant task fairness (`document_indexing_task.py#L192-L231`).
3. **Bounded user-programmable execution**: user-drawn graphs face call-depth caps (`workflow_entry.py#L172-L175`), step/time limits (`#L211-L215`), quota layers, sandbox and ssrf_proxy containers. The platform assumes hostile inputs *by construction* — rare and correct.
4. **Architecture enforced by tooling**: import-linter for layers, catalogs+overrides for supply chain, two type-checkers during migration, suppression ledgers. Governance that survives team turnover.
5. **Failure states in the data model**: `indexing_status` + `Document.error` (`indexing_runner.py#L58-L67`) make ops legible to end users, not just to log readers.

## Risks and tradeoffs (confirmed observations vs hypotheses)

**Confirmed by reading:**

- `App.is_agent` property mutates + commits (`model.py#L471-L484`) — hidden write on read path; transaction interference risk. (User-visible impact: unestablished.)
- Model properties running queries (`App.site`, `App.app_model_config`, `model.py#L441-L450`) — N+1 traps: any list serializer touching them issues per-row queries.
- `services/dataset_service.py` is 4090 lines; `save_document_with_dataset_id` spans ~300 lines with nested branching — change amplification and review blindness. The repo's own 800-line guidance (`api/AGENTS.md`) says split it.
- Dual query-key systems in the frontend during the typed-client migration (`use-apps.ts#L70-L78` invalidates both) — every mutation must remember both or a screen goes stale.
- Frontend SSE 401 retry lacks visible cap/backoff (`base.ts#L540-L545`).

**Hypotheses (would verify before acting):**

- Stale-`PARSING` documents with no sweeper if a worker dies post-commit/pre-terminal-status (check `api/schedule/`).
- Tenant-queue pull/re-enqueue crash window losing queued batches (`document_indexing_task.py#L214-L227`).
- Single `SECRET_KEY` for all JWT classes → rotation and blast-radius concerns.
- Service-API owner-identity fallback (`service_api/wraps.py#L104-L120`) may grant broader identity than endpoints need.

## "If I owned this for 3 months" — prioritized

| # | Change | Why now | Migration path | Test strategy |
| --- | --- | --- | --- | --- |
| 1 | Make `is_agent` pure; one-time backfill of legacy chat→agent-chat modes | Correctness smell on a hot read path; tiny blast radius | Add data migration; keep property reading only; deprecation window logging any would-have-flipped rows | Unit: property purity; migration test on fixture rows |
| 2 | Stale-indexing sweeper (or verify existing) | Ops pain: stuck "parsing" docs are user-visible | New scheduled task marking `PARSING` older than N hours ERROR with reason | Task unit test + fake clock |
| 3 | Split `dataset_service.py` along DocumentService/DatasetService/permission seams | Review safety; the file resists comprehension | Mechanical move, import-linter contract to prevent re-merge | Existing tests must pass unchanged — that's the point |
| 4 | Finish the frontend typed-client migration for the highest-traffic resources | Every half-migrated resource doubles invalidation surface | Resource-by-resource; delete legacy keys per resource; codemod list | Query-key unit tests + one stale-screen regression test each |
| 5 | Cap+backoff the SSE 401 retry; centralize refresh single-flight | Prevents pathological loops on auth outages | Small pure change in `base.ts` | Vitest with mocked 401 sequences |

Note what is *not* on the list: rewrites, framework swaps, "microservices." The codebase's macro-architecture is sound; the leverage is in seams, invariants, and finishing started migrations. Saying exactly that — with evidence — is what separates a mid-level answer from cosplay-senior.

## Variation stress-tests (whiteboard yourself)

1. **10× traffic on chat**: what saturates first? (Likely: model-provider limits, then Redis pub/sub fanout, then DB writes for messages. The web tier scales flat; Celery scales by queue.)
2. **Add hard multi-region**: which invariants break? (Redis event replay locality; vector-store residency; SECRET_KEY sharing.)
3. **Sell on-prem**: which extensions/config already make this feasible? (Factory + `is_enabled()` extension gating, storage abstraction, middleware compose.)

Drill: pick risk #1 or #2 from the hypotheses, spend 45 minutes verifying against the code, and move it to either "confirmed" or "discarded" with anchors. That motion — hypothesis → evidence — is the whole job.
