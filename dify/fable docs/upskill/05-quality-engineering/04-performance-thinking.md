# Performance Thinking

Rule zero: **measure first**. Every claim below is a *place to look*, not a diagnosis. The senior habit is attaching a number before an opinion.

## Performance domains in this repo

| Domain | Where it lives | First measurement tool |
| --- | --- | --- |
| Server latency | Flask views, service layer | request logs (`ext_request_logging`), OTel traces (`ext_otel`, trace headers `app_factory.py#L100-L118`) |
| DB | SQLAlchemy queries | SQLAlchemy echo / pg_stat_statements |
| Queue throughput | Celery `dataset`/`priority_dataset` | queue depth in Redis, task latency logs (`document_indexing_task.py#L113-L114` logs per-dataset latency) |
| LLM/provider | model invocations | per-node timing via workflow layers; provider dashboards |
| Render | React (chat re-renders per SSE chunk, workflow canvas) | React DevTools profiler |
| Network | SSE, JSON payload sizes | devtools; payload of `handleStream` events |
| Bundle | Next/vinext build | analyzer; `knip` for dead code |
| Memory | in-memory CSV/exports, big documents | process metrics (`ext_app_metrics`) |

## Likely hotspots, with anchors (hypotheses to measure, labeled)

1. **N+1 via model properties**: `App.site`, `App.app_model_config`, `App.tenant` each run a query per access (`api/models/model.py#L441-L468`). Any list endpoint serializing these fields does K extra queries per page. Find it: log SQL while hitting the apps list; count queries vs rows.
2. **Chat re-render pressure**: every SSE chunk mutates the tree and re-renders the chat (`chat/hooks.ts` stream callbacks). With long answers this is dozens of updates/second. Mitigations to evaluate: chunk batching (rAF), `React.memo` on message rows, virtualization (`@tanstack/react-virtual` is already in the catalog).
3. **Workflow canvas**: ReactFlow with many nodes + Zustand — selector discipline decides whether dragging one node re-renders all. Audit selectors in `web/app/components/workflow/nodes/`.
4. **Serial awaits in indexing**: the runner processes documents sequentially in a loop (`indexing_runner.py#L71`) and embedding calls batch at some size — measure segment throughput before parallelizing (provider rate limits may be the true ceiling; parallelism would just hit them faster).
5. **Hot-path synchronous writes**: token validation is cached with single-flight precisely because it's per-request (`service_api/wraps.py#L23`); any new per-request DB write (see Kata 6) recreates the problem.
6. **Unbounded queries**: any list endpoint without pagination; check message history loading for long conversations (memory + latency). The `Message` tree pattern makes "load whole conversation" tempting.

## How to find each classic problem here

- **N+1**: enable SQL logging locally; hit a list page; if query count scales with row count, you have it. Fix: eager-load/join or bulk-fetch (fake-code contrast #6).
- **Serial async**: search for `for … await`-shaped loops in services/runners; ask "is the next iteration data-dependent on the previous?"
- **Expensive renders**: React profiler while streaming a long answer; look for message-row renders ∝ chunks.
- **Oversized bundles**: check what heavyweights land in the entry chunk — the catalog carries `echarts`, `mermaid`, `katex`, `monaco`, `lexical`; each should be dynamically imported per feature (verify with the analyzer, don't assume).
- **Missing indexes**: every new WHERE combination on big tables (messages, workflow runs, segments) needs an EXPLAIN; the repo's convention of indexing `tenant_id` (`model.py#L394`) is the floor, not the ceiling.

## The reasoning template (use in interviews)

1. What's the user-visible symptom and budget? ("first token < 1s", "list < 300ms")
2. Which tier is the bottleneck? Measure, don't debate.
3. Cheapest fix first: cache > index > batch > parallelize > rearchitect.
4. What does the fix cost? (cache: staleness; index: write amplification; parallel: rate limits)
5. Regression guard: perf assertion or dashboard alert.

## Drill

Instrument (locally) the apps list endpoint: count SQL queries with echo on, page of 20 apps. Predict the count first, then measure. Rubric — Strong: your predicted mechanism (which property causes which query) matches the log, and you can state the fix + its risk in two sentences.
