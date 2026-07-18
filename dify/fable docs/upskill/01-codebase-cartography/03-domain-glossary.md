# Domain Glossary

Product nouns, where they live in code, and the near-synonyms that will trip you up.

| Term | Meaning | Code home |
| --- | --- | --- |
| **Tenant / Workspace** | The isolation unit. Every shared table carries `tenant_id`; every query must scope by it (`api/AGENTS.md`) | `Tenant` — `api/models/account.py#L240`; join table `TenantAccountJoin` |
| **Account** | A human console user (can belong to several tenants, one "current") | `api/models/account.py#L87` |
| **EndUser** | An anonymous or app-scoped consumer of a *published* app — not an Account | `api/models/model.py` (see `EndUser` usage in `service_api/wraps.py#L98-L103`) |
| **App** | A configured LLM application. `mode` ∈ chat, agent-chat, completion, advanced-chat, workflow… (`AppMode`, `api/models/model.py#L363`) | `App` — `api/models/model.py#L392` |
| **App Model Config** | Versioned config blob for classic (non-workflow) apps: prompt, model, tools | `AppModelConfig` — `api/models/model.py#L632` |
| **Workflow** | Node-graph program (draft vs published versions) powering advanced-chat/workflow apps | `api/models/workflow.py`; engine entry `api/core/workflow/workflow_entry.py` |
| **Graph engine / graphon** | External library executing the node graph | dep `api/pyproject.toml:46`; wired at `workflow_entry.py#L185-L197` |
| **Conversation / Message** | Chat persistence; messages form a tree (regenerate = sibling branch — see frontend `parent_message_id` in `chat/hooks.ts#L620-L668`) | `Conversation` L1044, `Message` L1393 in `api/models/model.py` |
| **Dataset (Knowledge Base)** | A collection of indexed documents for RAG. UI says "Knowledge"; code says `Dataset` — same thing | `api/models/dataset.py#L166` |
| **Document → Segment → Child chunk** | Indexing hierarchy: uploaded file → chunks stored per-row → optional finer chunks (parent-child index) | `Document` L487, `DocumentSegment` L833, `ChildChunk` L1047 in `api/models/dataset.py` |
| **Indexing technique** | `high_quality` (embeddings + vector store) vs `economy` (keyword table) | set in `dataset_service.py#L2012-L2047` |
| **Hit testing** | Try a retrieval query against a dataset from the UI | `api/services/hit_testing_service.py#L101` |
| **RAG pipeline** | Newer, workflow-shaped ingestion pipeline for knowledge | `api/core/rag/pipeline/`, `api/services/rag_pipeline/` |
| **Provider / Model provider** | An LLM vendor credential + its models, per tenant | `api/models/provider.py`, `api/core/provider_manager.py` |
| **Plugin** | Marketplace-installable extension (tools, models, datasources) run by the plugin daemon (docker service `plugin_daemon`) | `api/core/plugin/`, `api/services/plugin/` |
| **Tool** | A callable capability for agents/workflows (builtin, API-based, plugin, MCP) | `api/core/tools/` |
| **DSL** | YAML import/export format for whole apps | `api/services/app_dsl_service.py` |
| **Site / WebApp** | The public shareable chat page generated per app (`enable_site`, `App.site` property `model.py#L441-L443`) | `api/controllers/web/`, `web/app/(shareLayout)` |
| **Service API token** | `app-…` bearer token for the public REST API | `ApiToken`; validated in `service_api/wraps.py#L55-L120` |
| **Passport** | Short-lived JWT for web-app (shared page) sessions — *not* the console session | `api/libs/passport.py`; header set in `web/service/base.ts#L505` |
| **InvokeFrom** | Enum tagging who triggered generation: DEBUGGER (console preview), WEB_APP, SERVICE_API… drives draft-vs-published workflow choice (`app_generate_service.py#L407-L418`) | `api/core/app/entities/app_invoke_entities.py` |

## Confusing near-synonyms

- **Account vs EndUser vs current_user**: `current_user` (a `LocalProxy`, `api/libs/login.py#L105-L118`) may hold *either*. Controllers assert which one they expect (`completion.py#L172-L173`). Mixing them up is a classic bug source.
- **Dataset vs Document (SQL) vs Document (RAG)**: `models/dataset.py` `Document` is the DB row; `core/rag` also has a `Document` value object (chunk of text + metadata) — same word, different layer. Check imports.
- **App mode `chat` vs `agent-chat`**: an old chat app silently *becomes* agent-chat if its config enables agent mode — via the `App.is_agent` property which mutates and commits (`model.py#L471-L484`). Dispatch honors this at `app_generate_service.py#L121-L123`.
- **Workflow (model) vs WorkflowRun vs draft**: the `Workflow` row is a version of the graph; a `WorkflowRun` is one execution; the *draft* is a special always-editable version fetched for `InvokeFrom.DEBUGGER`.
- **console API vs service API vs web API**: three controller trees, three auth models, one domain. Prefix tells you which: `/console/api`, `/v1` (service), `/api` (web app).

Drill: pick five nouns and, for each, find one table, one service, and one controller that touch it (use `rg`). Self-grade — Strong: you can also name the tenant-scoping column on each table.
