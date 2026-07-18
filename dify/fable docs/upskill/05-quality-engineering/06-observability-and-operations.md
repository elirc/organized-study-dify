# Observability and Operations

The organizing question, per flow: **"How would I know this broke?"** — before users tell you.

## What exists (inventory)

| Signal | Mechanism | Anchor |
| --- | --- | --- |
| Structured logs | module loggers (`logging.getLogger(__name__)` — house rule, no `print`), request context init (`init_request_context`, `app_factory.py#L61-L62`) | `api/AGENTS.md` logging rules: tenant/app ids in context; warning=retryable, error=terminal |
| Traces | OpenTelemetry (`ext_otel`), trace ids echoed to clients as `X-Trace-Id`/`X-Span-Id` response headers (`app_factory.py#L100-L118`); `ObservabilityLayer` on workflow runs (`workflow_entry.py#L218-L220`); external trace id accepted inbound (`get_external_trace_id`, `completion.py#L167-L169`) | support can correlate a user report → trace |
| Metrics | `ext_app_metrics` | process/app metrics |
| Errors | Sentry (`ext_sentry`); `logger.exception` at terminal failures | e.g. `completion.py#L198-L200` |
| LLM-specific ops | `core/ops/` + `ops_trace_task.py` — per-app tracing integrations (Langfuse-style), configurable per app (`App.tracing` column, `model.py#L416`) | product-level observability, not just infra |
| User-visible state | status columns: `Document.indexing_status/error`, workflow run status | `indexing_runner.py#L58-L67` |
| Health | `ping.py` controller (console), `celery_healthcheck.py` | LB checks + worker liveness |

## "How would I know it broke?" per major flow

- **Chat generation**: error-rate on chat endpoints (Sentry + logs), provider-error taxonomy (each mapped exception in `completion.py#L179-L200` is a countable class — alert on `ProviderNotInitializeError` spikes ≠ `QuotaExceededError` spikes, different owners). First-token latency needs a client-side or trace-derived metric — check whether one exists before claiming coverage.
- **Workflow runs**: run status distribution (failed/paused/succeeded) queryable from DB; `ExecutionLimitsLayer` terminations are a distinct signal (users hitting bounds = product issue, not bug).
- **Indexing**: count of documents in each `indexing_status` over time; stuck-in-PARSING age (the missing sweeper from the critique would double as the alert); per-dataset latency logged at `document_indexing_task.py#L113-L114`; tenant queue depth in Redis.
- **Auth**: 401/403 rates per surface (console/service/web); refresh-token failures; license-check failures **force logout** (`app_factory.py#L82-L96`) — an enterprise-license service outage looks like a mass logout; know that before the incident.
- **Frontend**: Sentry (`@sentry/react` in catalog), plus the toast layer means many errors are *seen* by users but never *recorded* — check whether `fetch.ts#L61-L79` reports before it toasts (if not: gap worth a ticket).

## Deploy and rollback posture

Images built from `web/Dockerfile` and `api/Dockerfile` (Makefile build targets); full-stack compose in `docker/`. DB migrations run via `flask db upgrade` — the operational contract is: **migrations are additive and forward-only in practice**, code tolerates both schema versions during rollout (server defaults help — `model.py#L406-L424`). Rollback = redeploy previous image; schema rollback = roll forward with a fixing migration. Celery workers must be deploy-ordered with payload/schema compatibility in mind (old worker, new payload — see 02/03 type contracts).

## The gap analysis habit (portable)

For any feature you ship, fill this table before merge:

| Question | Answer for your change |
| --- | --- |
| What log line fires on the failure path, at what level, with which ids? | |
| Which existing dashboard/alert would move? | |
| Can support correlate a user report to a trace? (X-Trace-Id) | |
| Is failure user-visible somewhere honest (status, toast, error state)? | |
| What's the rollback story? | |

## Drill

Pick the tenant-isolated queue (`document_indexing_task.py#L192-L231`). Design its minimal dashboard: three numbers and one alert that would have caught the "stale flag" failure mode. Rubric — Strong: your alert is on a *symptom users feel* (queue age), not an internal detail (flag exists), and you can say why that ordering matters.
