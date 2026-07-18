# Refactor and Design Katas

Eight senior katas. Deliverables are *documents or diffs you don't merge* — the practice is the design reasoning. Self-grade with each kata's criteria; "pass" means you'd be comfortable defending the artifact to a staff engineer.

---

## K1: Fix the boundary leak (`App.is_agent`)
Task: produce the diff + migration plan making `is_agent` pure (`model.py#L471-L484`): backfill migration, pure property, transition logging for stragglers, and the PR description explaining risk.
Self-grade: ✅ if your plan handles rows that would have flipped *after* the backfill (in-flight configs) and states how you'd verify zero behavior change (query counts of each mode before/after).

## K2: Split `dataset_service.py` (4090 lines)
Task: propose the file split — new module boundaries, what moves where, import-linter contract to keep them apart. No code, just the map + sequencing (which split lands first and why).
Self-grade: ✅ if every proposed module has a one-sentence responsibility, no cyclic imports in your map, and the first PR is mechanically reviewable (<500 lines of *moves*).

## K3: Design the outbox RFC (S1's paper twin)
Task: write the RFC for transactional outbox per [07/02's template](../07-career-and-collaboration/02-writing-prs-and-rfcs.md), including the "do nothing" option honestly costed.
Self-grade: ✅ if a reader can *reject* the proposal from your own data (that's what an honest RFC enables).

## K4: Remove duplication without removing defense
Task: the billing checks exist in service (`dataset_service.py#L1982-L2006`) and task (`document_indexing_task.py#L64-L91`). Extract a shared checker that both call, preserving the trust-boundary re-validation. Diff it.
Self-grade: ✅ if the shared function is context-free (no flask, no current_user — it must run in workers) and each call site's failure handling stays distinct (HTTP error vs status-column write).

## K5: Type-safety kata: kill the kwargs injection
Task: design (not implement) a typed alternative to decorator kwargs injection (`service_api/wraps.py#L79-L99`): e.g., a typed request-context object or explicit dependency function. Weigh against Flask idiom churn.
Self-grade: ✅ if your design survives the question "what breaks for the 100 existing endpoints, and what's the incremental path?"

## K6: Migration design: message tree pagination
Task: conversations load messages; the tree (parent ids) makes "latest thread" queries awkward. Design the schema/index/query changes for O(page) thread loading — possibly a materialized `thread_root_id`/`path` column. Include backfill of millions of rows.
Self-grade: ✅ if the backfill is chunked, resumable, and the read path works mid-backfill (dual-read or feature-gate).

## K7: Reduce the N+1: apps list serialization
Task: diff a fix for the `App.site`/`app_model_config` property N+1 (`model.py#L441-L450`) on the list endpoint — bulk-load + pass-through, or serializer change. Measure plan included.
Self-grade: ✅ if you keep the properties working for single-object paths (blast radius!) and your measurement plan produces a before/after query count.

## K8: Review the flawed PR, fully
Task: take Kata 8 from [04/04-review-katas](../04-code-reading-gym/04-review-katas.md) (synchronous cascade delete) and write the complete review: findings *plus* the alternative design sketch *plus* the kind message that redirects the author without demoralizing them.
Self-grade: ✅ if a colleague reading only your review could implement the right solution, and the tone passes the "would I be glad to receive this?" test.

---

**Cadence**: one kata per week alongside real tickets. Write every artifact as if it will be read — because in interviews, you'll be reciting them.
