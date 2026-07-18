# Verification Log

Running record of what was inspected while authoring this curriculum. Date: **2026-07-10**. Environment: Windows 11, repo at `03 Master Projects/masterdidy/dify`, git repo present but **no commits on `main`** (fresh snapshot — `git log` fails with "does not have any commits yet"), so no git history was usable for archaeology.

## Commands run (all read-only)

| Command | Result |
| --- | --- |
| `ls` of repo root, `api/`, `web/`, `api/controllers/*`, `api/core/*`, `api/services/`, `api/models/`, `api/tasks/`, `api/extensions/`, `web/app/`, `web/service/`, `web/app/components/workflow/store/` | Structure mapped; recorded in 01-system-map |
| `git log --oneline` | Failed: branch has no commits (snapshot clone) |
| `cat` root `package.json`, `pnpm-workspace.yaml`, `Makefile`, `AGENTS.md`, `api/AGENTS.md`, `web/AGENTS.md` | Read fully |
| `grep -n "^class "` over `api/models/*.py` | Class → line map for models |
| `grep -n "^def \|^class "` over `api/controllers/console/wraps.py` | Decorator inventory |
| `grep -rn ssePost web/...` | SSE call sites found (chat hooks, debug, share services) |
| `find . -name "graphon*"` + `grep graphon api/pyproject.toml` | `graphon~=0.3.1` is an external PyPI dependency (`api/pyproject.toml:46`), not vendored |
| `ls docker/*.yaml`, service grep of `docker-compose.middleware.yaml` | Services: db_postgres, db_mysql, redis, sandbox, plugin_daemon, ssrf_proxy, weaviate |
| `ls api/migrations/versions | wc -l` | **171** Alembic migration files |

## Files read in full or in targeted ranges (line anchors confirmed at read time)

- `api/app_factory.py` (entire, 231 lines)
- `api/libs/login.py` (entire, 118), `api/libs/passport.py` (entire, 24)
- `api/controllers/console/app/completion.py` (entire, 224)
- `api/controllers/console/wraps.py` (L306-L340 + symbol map)
- `api/controllers/service_api/wraps.py` (L1-L120)
- `api/services/app_generate_service.py` (entire, 436)
- `api/services/account_service.py` (L195-L269 + symbol map)
- `api/services/dataset_service.py` (L1967-L2299 + task-dispatch grep; file is 4090 lines)
- `api/services/hit_testing_service.py` (symbol map + L101-L131)
- `api/tasks/document_indexing_task.py` (entire, 259)
- `api/core/indexing_runner.py` (L55-L169; file is 852 lines)
- `api/core/workflow/workflow_entry.py` (L1-L260)
- `api/models/model.py` (L392-L501 + class map; 2589 lines)
- `api/models/account.py` (L87-L261; 429 lines)
- `web/service/fetch.ts` (entire, 277), `web/service/base.ts` (L1-L220, L440-L580; 878 lines)
- `web/service/use-apps.ts` (L1-L90)
- `web/app/components/base/chat/chat/hooks.ts` (L620-L709, L1100-L1230; 1286 lines)

## Commands **not** run (labeled __inferred__ wherever cited)

- `make dev-setup`, `flask run`, `celery worker`, `pnpm dev` — starting long-running services is forbidden by `api/AGENTS.md` ("Never start long-running services as part of agent work") and unnecessary for doc authoring.
- `make test`, `pnpm type-check`, `pnpm lint` — not executed; the dependency install (uv sync + pnpm install) had not been performed in this snapshot and would mutate the working tree. All test/lint/typecheck commands cited are transcribed from `Makefile` L66-L105 and root `package.json` scripts.

## Uncertainties and areas not covered

- **graphon internals**: the workflow graph engine (`GraphEngine`, `VariablePool`, node scheduling) is an external package; claims about it are limited to its call-sites in `api/core/workflow/workflow_entry.py`. Its behavior beyond that is labeled inferred.
- **dify-agent/** and `api/enterprise/`: not covered; enterprise-gated branches (license checks, billing internals) are described only where they intersect mainstream flows.
- `docs-codex-learning/`: pre-existing generated learning docs from another tool; not used as a source, not overwritten.
- Exact frontend test-runner invocation (`pnpm --filter dify-web test …`) is inferred from workspace layout; `web/package.json` scripts were not read line-by-line. Learners should run `pnpm run` in `web/` to list scripts.
- `App.is_agent` mutating `mode` + committing inside a property getter (`api/models/model.py#L471-L484`) is flagged as a **confirmed code smell** (verified by reading), but whether it causes user-visible bugs was not established — treated as a teaching seam / "investigate," not a bug claim.
- Line anchors were captured from reads on 2026-07-10 against this exact working tree; the tree has no commit hash to pin (no commits), so anchors cannot be tied to a revision.
