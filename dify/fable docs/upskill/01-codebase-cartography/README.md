# 01 — Codebase Cartography

Before you can change a system safely you must know where things live, who owns what, and how the real flows run. This module builds that map for Dify and — more importantly — teaches the *method* so you can map any repo in a day.

| File | What you get |
| --- | --- |
| [01-system-map.md](01-system-map.md) | Monorepo shape, ownership map, public vs private surfaces |
| [02-file-reading-order.md](02-file-reading-order.md) | The 30 files to open, in order, with junior/mid/senior paths |
| [03-domain-glossary.md](03-domain-glossary.md) | Product nouns → code locations; confusing near-synonyms |
| [04-runtime-and-tooling-map.md](04-runtime-and-tooling-map.md) | Build/test/lint toolchain, runtime boundaries, env vars |
| [05-key-flows.md](05-key-flows.md) | Six end-to-end flow traces with drills — the module's core |

**Method to steal for any repo**: read the root metadata first (manifests, Makefile, AGENTS/CONTRIBUTING), list the top two directory levels, find the app factory / entry point, then trace one write path end to end before reading anything "interesting." Interviewers asking "how do you ramp up on a new codebase?" want exactly this answer, with an example — use this repo as yours.
