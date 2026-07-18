# Side Effects, Async, and Reliability

## Map of side effects

| Side effect | Where triggered | Mechanism |
| --- | --- | --- |
| Document indexing (vector writes) | `dataset_service.py#L2271-L2277` | Celery, queues `dataset` / `priority_dataset` |
| Workflow execution | `app_generate_service.py#L171-L174` | Celery task + Redis event stream |
| Email (invites, resets, codes) | `api/tasks/mail_*.py` (a dozen tasks) | Celery `mail` queue |
| LLM/provider calls | `core/model_manager.py`, app runners | Outbound HTTPS |
| User-controlled fetches (website crawl, remote files) | extractors / tools | Must route via `core/helper/ssrf_proxy` (rule in `api/AGENTS.md`) |
| Webhooks/triggers | `api/controllers/trigger/`, `core/trigger/` | Inbound + scheduled |
| Cleanup (delete dataset/document/conversation) | `tasks/clean_*.py`, `delete_*` tasks | Celery, idempotent-by-design requirement |
| Scheduled jobs | `api/schedule/` | Celery beat–style entries |
| Ops tracing | `tasks/ops_trace_task.py` | async telemetry export |

Rule from the repo (`api/AGENTS.md`): "Background tasks that touch storage must be idempotent, and should log relevant object identifiers." Treat that line as the grading rubric for any task you write.

## The reliability vocabulary, each with a local example

- **Idempotency** — safe to run twice. Indexing re-queries current status and rebuilds segments; `run_in_splitting_status` even deletes existing segments before re-creating (`indexing_runner.py#L129-L158`). When you retry a failed document, you don't get doubled chunks.
- **Retries** — mostly *manual/explicit* here (e.g., `retry_document_indexing_task` re-enqueued from a user action, `dataset_service.py#L1934`) rather than blanket Celery auto-retry. Tradeoff: no retry storms, but transient failures need a human or a sweep job. Contrast in interviews with auto-retry + backoff + max-attempts.
- **Ordering/fairness (backpressure-ish)** — the tenant-isolated queue (`document_indexing_task.py#L192-L231`): while one batch runs, further batches for that tenant park in a Redis list; on completion the worker pulls the next `TENANT_ISOLATED_TASK_CONCURRENCY` items and re-enqueues. One noisy tenant can't monopolize the shared `dataset` queue. This is a hand-rolled fairness scheduler — rare to see written out; learn it.
- **Timeouts / execution limits** — the graph engine gets `ExecutionLimitsLayer(max_steps, max_time)` (`workflow_entry.py#L211-L215`) and a call-depth cap (`#L172-L175`). Every unbounded loop a user can draw must hit a wall you chose.
- **Failure visibility** — errors are *persisted where users look*: `Document.error` + `indexing_status=ERROR` (`indexing_runner.py#L58-L67`, `document_indexing_task.py#L81-L91`). Logs are for engineers; status columns are for users. Both are observability.
- **Compensation** — quota is reserved before work and *refunded* on failure (`app_generate_service.py#L107-L112`, `#L264-L267`). Reserve→commit→refund is a mini saga; name it when asked about "distributed transactions lite."
- **Delivery semantics** — SSE events via Redis: Streams = durable, replayable (client can reconnect and catch up); Pub/Sub = at-most-once, so task start is gated on first subscriber with a 200ms fallback timer (`app_generate_service.py#L41-L85`). Choosing semantics per transport, in code, is a textbook you can point at.

## Risky spots to keep an eye on (labeled *investigate*)

1. **Commit-then-enqueue gap** (Flow 3): process death between commit and `.delay()` leaves rows in `PARSING`-less limbo (status still waiting). A sweep job or outbox would close it. Check whether `schedule/` already sweeps stale documents before proposing.
2. **Tenant-queue flag protocol**: if a worker dies after `pull_tasks` but before `apply_async` (`document_indexing_task.py#L214-L227`), pulled items are lost from the Redis list. Quantify likelihood before calling it a bug — but be able to *design* the fix (peek-then-ack).
3. **Recursive SSE retry on the frontend** (`base.ts#L540-L545`) — no backoff/cap visible at call site.
4. **Chained summary task** failures are logged and swallowed (`document_indexing_task.py#L167-L172`) — acceptable (non-critical enrichment), but note the deliberate decision: chained ≠ transactional.

## Fake-code contrast

```python
# Illustrative fake code: not from this repo
# BAD — side effect inside the transaction; retry double-sends
with session.begin():
    order.status = "paid"
    send_receipt_email(order)   # crashes => rollback, but email already sent

# BETTER — commit state, then enqueue idempotent task with an id
with session.begin():
    order.status = "paid"
send_receipt_task.delay(order_id)  # task re-reads state, dedupes by order_id
```

Real analogue: commit at `dataset_service.py#L2269`, enqueue at `#L2271-L2277`, re-validate inside the task.

## Drills

1. Design (paper) the outbox version of document-indexing dispatch: table columns, the poller, idempotency key, and what happens on poller crash. Strong = you can argue when commit-then-enqueue is *good enough* (it usually is at this scale) — engineering is calibrated paranoia.
2. Read `TenantIsolatedTaskQueue` (`core/rag/pipeline/queue.py`) and write its state machine (flag set/cleared, list push/pull). Find one interleaving you can't prove safe.
3. For each row of the side-effects map, answer "how would I know it broke?" — cross-check against [05-quality-engineering/06-observability-and-operations.md](../05-quality-engineering/06-observability-and-operations.md).

## Interview angle

- "Exactly-once processing?" → doesn't exist; you get at-least-once + idempotency or at-most-once + gaps; point at the Streams-vs-PubSub branch as code that *chooses*. (→ 08/04)
- "One customer floods your queue — what do you do?" → tenant-isolated queue walkthrough. (→ 08/04-system-design variation)
- "How do you handle partial failure across DB + external system?" → reserve/commit/refund quota saga + status-column reconciliation. (→ 08/03, 08/04)
