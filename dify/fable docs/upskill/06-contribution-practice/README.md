# 06 — Contribution Practice

Reading builds judgment; shipping builds proof. This module is a ladder of realistic work items against this codebase. All of them respect **contribution realism**: each states why it's useful, why the blast radius is small, which existing pattern it follows, how it's tested, and what would make a maintainer reject it.

| File | Contents |
| --- | --- |
| [01-good-first-tickets.md](01-good-first-tickets.md) | 16 junior tickets across the repo |
| [02-mid-level-feature-tickets.md](02-mid-level-feature-tickets.md) | 10 cross-layer tickets requiring design notes |
| [03-senior-build-projects.md](03-senior-build-projects.md) | 6 multi-week projects |
| [04-refactor-and-design-katas.md](04-refactor-and-design-katas.md) | 8 senior katas with self-grading |

Every item carries an **Interview story potential** line — completing it should produce a STAR story ([08/06](../08-interview-prep/06-behavioral-star-stories.md)). Do the work, then immediately write the story while details are fresh.

Workflow reminders: branch per ticket; `make format lint type-check test` before any backend PR; `pnpm lint:fix && pnpm type-check` + vitest for web; PR format per [07/02](../07-career-and-collaboration/02-writing-prs-and-rfcs.md).
