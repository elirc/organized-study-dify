# 02 — Stack and Language Mastery

Dify runs two stacks side by side: TypeScript/React on the client, Python/Flask/Celery on the server. That's a gift for a JS-native learner: the TS half deepens what you know; the Python half teaches you to separate *concepts* (event loops, contracts, layering) from *syntax* — the exact skill interviewers probe with "you've mostly done JS; how would you handle our Python service?"

| File | Covers |
| --- | --- |
| [01-language-runtime-model.md](01-language-runtime-model.md) | JS event loop & async in the browser code; Python threading/workers on the server |
| [02-framework-mental-models.md](02-framework-mental-models.md) | React 19 rendering, TanStack Query, Zustand; Flask + Celery from first principles |
| [03-type-system-and-contracts.md](03-type-system-and-contracts.md) | TS strict-mode techniques; Pydantic v2 + SQLAlchemy typed models; contracts across the wire |
| [04-tooling-and-build-system.md](04-tooling-and-build-system.md) | pnpm workspaces + catalogs, vite-plus/vinext, uv, ruff/mypy/pyrefly |

Rule for the whole module: every concept anchors to a real file. If you can't point at where this repo exercises an idea, you don't get to claim it in an interview.
