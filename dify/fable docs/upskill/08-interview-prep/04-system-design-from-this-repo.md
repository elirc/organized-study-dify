# System Design From This Repo

**The exercise**: "Design a platform where teams build and publish LLM chat apps with knowledge-base retrieval." You will reverse-engineer Dify on a whiteboard, step by step, comparing your moves to what the repo actually chose. Practice this full walkthrough aloud at least twice before an interview.

---

## Step 1 — Requirements (5 min)

Functional: workspaces with members/roles; build apps (prompt/config or node graph); test in console; publish (share page + REST API); chat with streaming; upload documents → retrieval; usage limits/billing hooks.
Non-functional: multi-tenant isolation; long-running generations (seconds–minutes); document indexing (minutes); streaming UX; provider outages happen; per-tenant fairness.

*Junior* answers list features. *Mid* separates functional/non-functional and names tenancy early. *Senior* asks scale/SLA questions first (how many tenants? doc sizes? concurrent generations?) and states the two hard problems up front: **long-running work over HTTP** and **isolation everywhere**.

## Step 2 — API sketch (10 min)

Three surfaces (what Dify chose — `api/controllers/{console,service_api,web}`):
- Console API: session+CSRF, powers the dashboard.
- Service API `/v1`: bearer app tokens for customers (`service_api/wraps.py#L55-L120`).
- Web API: passport JWT for published share pages.

Core endpoints: `POST /apps/<id>/chat-messages` (streaming flag), `POST …/stop`, datasets/documents CRUD, hit-testing, publish.

*Mid*: separates surfaces by auth model. *Senior*: declares contract strength per surface (service API sacred; console co-versioned with the UI) and long-op semantics (SSE + replayable run ids) — anchor `app_generate_service.py#L153-L185`.
Simpler alternative worth naming: one surface + scopes — fine for a small product; Dify's split earns its cost via different auth *and* different compatibility promises.

## Step 3 — Data model (10 min)

Draw: Tenant ─< Account (join w/ role); Tenant ─< App ─< {AppModelConfig|Workflow versions, Site, Conversation ─< Message(tree)}; Tenant ─< Dataset ─< Document ─< Segment ─< ChildChunk; ApiToken; Provider.
Dify's actual columns to cite: `tenant_id` indexed on every table (`model.py#L394`); versioning-by-pointer for configs/workflows (`model.py#L404-L405`); message tree via parent ids; status machines on Document.

*Mid*: correct entities + tenancy column + one index argument. *Senior*: versioning-by-pointer for instant rollback, JSON-vs-table calls (`retrieval_model`), and the message-tree pagination cost.

## Step 4 — The two engines (10 min)

**Generation path**: request → quota reserve → rate limit → mode dispatch → (streaming) enqueue Celery task, HTTP subscribes to Redis event stream keyed by pre-minted `workflow_run_id`; replay on reconnect. Anchors: `app_generate_service.py#L87-L270`, subscribe gating `#L41-L85`, engine limits `workflow_entry.py#L172-L215`.
**Indexing path**: upload → commit rows → enqueue (ids only) → phased status transactions → extract/transform/load into vector store → status/error persisted; per-tenant queue fairness. Anchors: Flow 3.

*Junior*: draws one box "backend." *Mid*: separates web/worker tiers, names the queue, streams events. *Senior*: delivery-semantics choice (streams vs pub/sub — the code literally branches, `#L65-L76`), bounded execution for user-programmable graphs, and the trust-boundary re-validation in tasks.

## Step 5 — Scaling and failure (10 min)

- Web tier stateless → horizontal; workers scale per queue; Redis is the coupling point (broker + events + cache) — its failure story is *the* question to volunteer.
- Provider outages: error taxonomy per provider exception (`completion.py#L186-L196`), quota refunds on failure.
- Noisy tenants: app-level active-request caps (`#L272-L291`), tenant-isolated indexing queues.
- Stuck work: status columns + (proposed) sweeper — honest gap discussion earns points.

*Senior additions*: capacity math sketch (messages/sec × payload → DB write budget; embeddings throughput as indexing ceiling), and blast-radius ordering: what you shed first under load (indexing before chat).

## Step 6 — Tradeoffs recap (say this unprompted)

1. Shared-table tenancy + discipline vs hard isolation: chose cheap + codified rules + fairness at queues.
2. SSE + Celery + replay vs holding requests: chose complexity in infra to buy UX + reconnect.
3. Commit-then-enqueue vs outbox: chose simple with a small crash window (name the window!).
4. Three API surfaces vs one: chose per-audience contracts at the price of duplication.

---

## Variation prompts (practice each for 10 minutes)

1. **"Now make it real-time collaborative"** (two editors, one workflow): CRDT or OT for the graph (`loro-crdt` is already in the catalog — the repo is heading there), presence via Socket.IO (`ext_socketio` exists), conflict story for publish.
2. **"10× chat traffic"**: what saturates? (provider limits → queue Redis fanout → message-row writes). Mitigations in order: provider pooling/load-balancing (`model_load_balancing_service.py` exists!), Redis cluster/streams partitioning, message write batching.
3. **"Add strict data residency per tenant"**: storage + vector store per-region bindings (collection binding pattern generalizes), key management, where shared-table tenancy finally breaks.
4. **"Support 100MB documents"**: chunked upload, streaming extraction, indexing checkpoints (status machine already supports resume-in-splitting — `indexing_runner.py#L129+`), cost controls.
5. **"On-prem, air-gapped"**: extension `is_enabled()` gating (`app_factory.py#L207-L213`), provider abstraction, license checks — much of the machinery exists; the exercise is knowing which cloud assumptions to sever.

## Grading yourself

Record the walkthrough. Pass bar (mid): all six steps hit, tenancy never forgotten, one tradeoff per step. Strong: you volunteered failure modes before being asked, cited at least four real anchors, and your variation answers reused mechanisms already in the codebase instead of inventing new ones — that reuse instinct reads as experience.
