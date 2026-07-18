# Tooling and Build System

Tooling literacy is quiet seniority: knowing *why* a lockfile, a catalog, or a type-checker is configured the way it is.

## Frontend workspace

- **pnpm 11 + workspaces**: members `web`, `e2e`, `sdks/nodejs-client`, `packages/*` (`pnpm-workspace.yaml`). `packages/dify-ui` is the design system, consumed as `@langgenius/dify-ui` — UI primitives are versioned *inside* the monorepo, so a breaking button change shows up in the same PR that uses it.
- **Catalogs**: every dependency version is pinned once in the `catalog:` section of `pnpm-workspace.yaml`; packages reference `"catalog:"`. Transferable concept: single source of truth for versions across a monorepo (same job as syncpack/renovate groups). Also note `overrides:` force-patching transitive deps (security floors for `dompurify`, `esbuild`…) and supply-chain controls (`minimumReleaseAge`, `trustPolicy: no-downgrade`, `allowBuilds: false` for native postinstalls) — real-world supply-chain defense you can cite.
- **vite-plus / vinext**: `vite` and `vitest` are *aliased* to `@voidzero-dev/vite-plus-*` packages, and dev runs through `vinext` + a dev proxy (root `package.json` scripts: `dev`, `type-check`, `lint`). Lesson: read the overrides before assuming stock behavior of any tool.
- **ESLint 10 (antfu config) + tsslint**, `knip` for dead code, Storybook 10 for components, Playwright/Cucumber in `e2e/`.

## Backend toolchain

- **uv** for env + deps (`uv sync --dev`, `uv run --project api …`); lockfile `api/uv.lock`. Think "pnpm for Python."
- **ruff** = ESLint+Prettier in one (`make format`, `make lint` — Makefile L66-L83); plus **import-linter** (layer boundaries as lint rules — architecture enforced by CI, not convention) and dotenv-linter.
- **Two type-checkers**: pyrefly and mypy both run in `make type-check` (L85-L89). Redundancy is deliberate during migration between checkers — a live example of "how do you adopt stricter typing in a big codebase without stopping the world."
- **pytest** with xdist (`make test`, L97-L105); unit tests local, integration tests CI-only (root `AGENTS.md`).
- **Alembic via Flask-Migrate**: 171 versioned migrations (`api/migrations/versions/`); apply with `flask db upgrade`. `create_migrations_app` builds a minimal app for migration runs (`api/app_factory.py#L222-L230`) — migrations shouldn't boot Celery/Redis/Sentry.

## Docker / dev topology

`make dev-setup` starts only **middleware** (`docker/docker-compose.middleware.yaml`: postgres L3, mysql L43, redis L83, sandbox L105, plugin_daemon L131, ssrf_proxy L197, weaviate L226) while API/web run on your host — fast iteration with real infra. Full-stack compose (`docker/docker-compose.yaml`) is for deployment-shaped runs. Transferable: "middleware-in-docker, code-on-host" is the standard dev topology for polyglot monorepos.

## What a senior inspects first in any repo's tooling

1. Lockfiles + version pinning strategy (here: catalogs, exact saves).
2. What CI actually runs (`.github/workflows/` — compare against Makefile; if they diverge, CI wins).
3. Where boundaries are *enforced* (import-linter; eslint no-barrel-files) vs merely documented.
4. The escape hatches (`eslint-suppressions.json`, `pyrefly-local-excludes.txt`) — tech-debt ledgers in disguise. Their size ≈ honesty of the codebase.

## Drills

1. From `pnpm-workspace.yaml`, explain in two sentences what happens today when a new transitive dep ships a malicious postinstall script. (Strong: cite `allowBuilds`/`minimumReleaseAge`.)
2. Open `api/.ruff.toml` (or the pyproject section) and find one enabled rule family you don't know; write a one-line justification for it.
3. Run `pnpm run` in `web/` and map each script to its Makefile-equivalent concept on the API side.

## Interview angle

1. "Monorepo vs polyrepo?" → answer with catalogs, shared design system in-repo, atomic cross-package PRs; costs: tooling complexity (vite-plus aliasing). (→ 08/04)
2. "How do you roll out a stricter type-checker/linter on a legacy codebase?" → two-checkers-in-parallel + suppression ledgers, from this repo. (→ 08/01)
3. "What's your migration story?" → Alembic discipline: minimal migrations app, upgrade path, 171-file history. (→ 08/03)
