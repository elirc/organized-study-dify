# Testing Strategy

## The repo's layers

| Layer | Where | Runs | Covers |
| --- | --- | --- | --- |
| Backend unit | `api/tests/unit_tests/` (mirrors source tree: controllers, services, core, tasks, libs, models, …) | locally, `make test` (pytest + xdist) | pure logic, service behavior with mocked I/O |
| Backend integration | `api/tests/integration_tests/` | **CI-only** (root `AGENTS.md`: "not expected to run in the local environment") | real vector stores, providers, storage |
| Container integration | `api/tests/test_containers_integration_tests/` | CI | dockerized dependencies |
| Frontend unit/component | `web/__tests__/`, colocated `*.spec.ts(x)` (e.g. `web/service/base.spec.ts`, `chat/__tests__/hooks.spec.tsx`) | vitest (see `web/docs/test.md` — the repo's canonical testing instructions) | hooks, services, components |
| E2E | `e2e/` | Playwright + Cucumber | user journeys |

House rules that shape tests: TDD red→green→refactor and Arrange-Act-Assert (root `AGENTS.md`); frontend tests must follow `web/docs/test.md` *and* the `frontend-testing` skill (`web/AGENTS.md`).

## What belongs at which layer (the transferable judgment)

- **Unit-test the decisions, not the wiring**: `_get_max_active_requests` (`app_generate_service.py#L272-L291`) — min-of-limits with 0-means-infinite — is a perfect unit test target: pure, branchy, cheap. The Celery wiring around it is not.
- **Mock at boundaries you own, fake at boundaries you don't**: frontend tests mock `ssePost` (`chat/__tests__/hooks.spec.tsx#L11`, `#L154+`) — the service layer is the seam, so component tests never touch the network. Backend service tests mock providers/vector stores.
- **Don't unit-test the framework**: no tests assert that Flask routes route or Pydantic validates types. Test *your* validators (e.g., the UUID validator in `ChatMessagePayload`, `completion.py#L60-L65`).
- **Integration tests earn their cost only where mocks would lie**: vector-store semantics, provider quirks, migrations — hence CI-only.

## Fixtures, isolation, and flake prevention

- Shared fixtures: `api/tests/conftest.py` + per-tree conftests; `api/tests/fixtures/` for data; `workflow_test_utils.py` for graph tests.
- **Time**: code uses `naive_utc_now()` (`libs/datetime_utils.py`) rather than raw `datetime.now()` — a single seam to freeze in tests. Frontend: fake timers for the 200ms-fallback-style logic.
- **Randomness/ids**: batch ids use `time.strftime + secrets.randbelow` (`dataset_service.py#L2059`) — tests should assert *shape*, not value.
- **Isolation**: unit tests must not need Redis/Postgres; anything needing brokers goes to integration. The tenant-queue logic is the hard case — its Redis protocol is exactly the kind of thing to wrap behind a class (`TenantIsolatedTaskQueue`) so a fake can stand in.
- **Flake sources to preempt here**: SSE callback ordering, Celery eager-mode differences, `Date.now()` placeholder ids in chat tests (two calls in the same ms).

## What NOT to test

Generated clients, third-party engine internals (graphon), styling, i18n string content, and anything the type-checker already proves. Every test is a maintenance liability; spend the budget on invariants (tenant scoping, status machines, quota math).

## Drill

Pick `AppGenerateService._build_streaming_task_on_subscribe` (`#L41-L85`). Write (on paper) the test list you'd ship: happy path per transport, double-subscribe, timer-fires-first, enqueue-raises. For each: what do you fake (config, timer, task), and what do you assert? Rubric — Strong: you replace the real `threading.Timer` (inject or patch) rather than sleeping in tests.

## Interview angle

- "How do you decide unit vs integration?" → mocks-would-lie criterion + this repo's CI-only integration split. (→ 08/05)
- "How do you make async code testable?" → seams: inject timers/queues, ids-not-objects, status columns you can assert on. (→ 08/01, 08/03)
