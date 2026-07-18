# API Design and Data Modeling Cards

## Q1: Design a REST API for a resource with long-running operations.
Anchor: chat/workflow generation — POST returns SSE stream or task handle; stop endpoint (`completion.py#L203-L224`); event replay by run id (Flow 2).
Mid: 202-style patterns, status polling vs SSE, stop as its own resource.
Senior: pre-minted operation ids for idempotent subscription; delivery decoupled from execution.

## Q2: How do you version / evolve a public API without breaking clients?
Anchor: three surfaces with different contracts (console vs service `/v1` vs web); additive-change rule; SDKs in `sdks/` as the blast-radius proof.
Mid: additive fields, deprecation windows, versioned paths for breaking changes.
Senior: *classify surfaces by contract strength* — the repo treats console API as quasi-internal, service API as sacred; that classification is the real skill.

## Q3: Where does validation belong?
Anchor: the validation onion ([03/03](../03-architecture-and-patterns/03-validation-auth-and-permissions.md)); queue-boundary re-validation (`document_indexing_task.py#L64-L91`).
Mid: every trust boundary once; controller = shape, service = business, DB = last resort.
Senior: the queue as a trust boundary (time passes) — most candidates never say this; you can.

## Q4: Authn vs authz — implement role-based access.
Anchor: JWT+refresh (`account_service.py#L200-L243`, Redis `#L150-L159`); role decorators (`console/wraps.py#L306-L337`); tenant roles (`account.py#L19`).
Mid: authn identifies, authz permits; roles per workspace, checked at the route + resource scoped in the query.
Senior: revocation (Redis-backed refresh), the single-SECRET_KEY tradeoff, and 404-vs-403 information leakage policy.

## Q5: What's IDOR? Show how you systematically prevent it.
Anchor: tenant-scoping rule (`api/AGENTS.md`), scoped lookups (`document_indexing_task.py#L82-L84`), `DatasetPermission` ACLs.
Mid: never trust ids; every query carries the caller's scope; test per resource (recipe 4).
Senior: making it *unforgettable* — codified rule, review checklist, scoped-loader helpers, lint/CI ideas.

## Q6: Design the schema for chat with regenerate/branching.
Anchor: `Conversation` (L1044) / `Message` (L1393) with parent linkage; frontend tree (`chat/hooks.ts`).
Mid: messages as tree via `parent_message_id`; sibling = regenerate; thread = root-to-leaf path.
Senior: query cost of trees (K6 kata), pagination strategy, and when to materialize paths.

## Q7: Transactions: when do you need one, and what must never be inside?
Anchor: two-phase indexing (`document_indexing_task.py#L93-L120`); commit-then-enqueue (`dataset_service.py#L2269-L2277`).
Mid: multi-row invariants need tx; network calls never inside; keep them short.
Senior: outbox pattern by name; saga/compensation (quota reserve-refund `app_generate_service.py#L107-L120`, `#L264-L267`).

## Q8: How do you paginate correctly?
Anchor: list endpoints in `controllers/console/app/` and service API (M4 ticket).
Mid: offset (simple, drifts under writes) vs keyset (stable, needs sortable unique key); envelope consistency.
Senior: pagination as a public contract — changing default page size is a breaking change; tie to Q2.

## Q9: Schema migration on a live system — walk me through it.
Anchor: 171 Alembic migrations; server defaults on `apps` (`model.py#L406-L424`); minimal migrations app (`app_factory.py#L222-L230`).
Mid: additive → dual-write/read → backfill → enforce → cleanup; never rename in place.
Senior: worker/deploy ordering with queued payloads (old worker, new schema), and index builds as their own risk.

## Q10: When is a JSON column right vs a proper table?
Anchor: `Dataset.retrieval_model` JSON vs `DatasetPermission` rows (`dataset.py`).
Mid: JSON for write-mostly evolving config you never query by; tables for anything filtered/joined/enforced.
Senior: migration escape plan for when the JSON grows query needs (expression indexes, extraction migration).

## Q11: Design multi-tenancy. Shared tables, schema-per-tenant, or DB-per-tenant?
Anchor: this repo = shared tables + `tenant_id` everywhere + indexed (`model.py#L394`) + task-queue fairness (`document_indexing_task.py#L192-L231`).
Mid: shared tables cheapest; isolation via disciplined scoping; noisy-neighbor handled at the queue tier.
Senior: when to graduate tiers (compliance, scale), and *non-DB* tenancy surfaces people forget: queues, caches, rate limits, vector stores (collection binding per dataset).

## Q12: How do you keep a cache (or second store) consistent with the DB?
Anchor: token cache with single-flight (`service_api/wraps.py#L23`); vector store reconciled via status columns (Flow 3); frontend query invalidation (Flow 6).
Mid: invalidation-on-write, TTL as backstop, stampede protection.
Senior: consistency *budget* per data class — auth tokens (seconds matter) vs app lists (minutes fine) — and revocation as the hard case.

---

**Cram usage**: Q3, Q4, Q5, Q7, Q9 recur constantly. Rehearse each with its anchor sentence: "In Dify, …" followed by the file you'd point at on a screen share.
