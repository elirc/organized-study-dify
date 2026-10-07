# Command Cheatsheet

All commands **inferred** from `Makefile`, root `package.json`, and `AGENTS.md` files (none executed during curriculum authoring — see [verification-log.md](verification-log.md)). Verify script names with `pnpm run` (web) and `make help` before relying on them.

## Bootstrap

| Task | Command | Source |
| --- | --- | --- |
| Full dev setup (middleware + envs + deps + migrate) | `make dev-setup` (in this filtered study copy, first restore `docker/envs/middleware.env.example`, `api/.env.example` and `web/.env.example` from upstream; they were not exported) | Makefile |
| Middleware only | `cd docker && docker compose -f docker-compose.middleware.yaml --env-file middleware.env -p dify-middlewares-dev up -d` | Makefile `prepare-docker` |
| Backend deps | `cd api && uv sync --dev` | Makefile `prepare-api` |
| Frontend deps | `pnpm install` (repo root) | Makefile `prepare-web` |
| Tear down dev volumes | `make dev-clean` (destructive!) | Makefile |

## Run

| Task | Command |
| --- | --- |
| API dev server | `uv run --project api flask run --host 0.0.0.0 --port 5001 --debug` |
| Celery worker | `uv run --project api celery -A app.celery worker -Q dataset,generation,mail,ops_trace` (queue list: check `celery_entrypoint.py` / task decorators) |
| Web dev server | `pnpm dev` (root; runs vinext dev + proxy concurrently) |

## Quality gates

| Task | Backend | Frontend (root) |
| --- | --- | --- |
| Format | `make format` | `pnpm lint:fix` |
| Lint | `make lint` (ruff + import-linter + dotenv-linter) | `pnpm lint` |
| Type check | `make type-check` (pyrefly + mypy) | `pnpm type-check` |
| All tests | `make test` | vitest via web package scripts (`pnpm run` in `web/` to list) |
| One test dir/file/case | `make test TARGET_TESTS=./api/tests/unit_tests/<path>` (append `::test_name` for one case) | `pnpm test <pattern>` in `web/` |

## Migrations

| Task | Command |
| --- | --- |
| Apply | `uv run --project api flask db upgrade` |
| New migration | `uv run --project api flask db migrate -m "add xyz"` then **hand-review** the generated file |
| History | `uv run --project api flask db history` |

## Docker images

| Task | Command |
| --- | --- |
| Build both | `make build-all` |
| Build one | `make build-api` / `make build-web` |

## Handy inspection one-liners (verified during authoring)

```bash
rg --files api | wc -l                          # backend file count
rg -n "class \w+\(.*Base\)" api/models          # model inventory
rg -n "@shared_task" api/tasks                  # celery task inventory
rg -n "console_ns.route|service_api_ns" api/controllers | head -50   # route inventory
ls api/migrations/versions | wc -l              # migration count (171 at authoring)
```
