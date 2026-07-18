# Codex Learning Documentation for Dify

This folder is a training layer over the Dify codebase. It does not replace the official docs; it teaches you how to read this repo like an engineer who can trace behavior, critique design, debug failures, and plan safe changes.

## The Four Suites

- `architectural-cartographer/` is the top-down map: entry points, folders, flows, architecture, critique, debugging, and review habits.
- `mission-learning-path/` is the active training campaign: ordered missions that force you to trace real code and self-grade your understanding.
- `user-story-build-path/` is the ticket simulator: ten realistic feature plans you can implement yourself without the docs giving away the code.
- `technology-best-practices/` is the stack guide: how this repo uses React, TypeScript, Next.js, React Query, Zustand, Flask, Pydantic, SQLAlchemy, Celery, Redis, Docker, and tests in real files.

## Recommended Reading Order

1. Read `architectural-cartographer/00-reading-map.md`.
2. Work through `architectural-cartographer/01-junior-engineer.md`.
3. Do missions 1-8 in `mission-learning-path/01-tier-1-junior-missions.md`.
4. Read `technology-best-practices/01-technology-map.md`.
5. Pick Story 1 from `user-story-build-path/01-stories.md` and implement it on your own branch.
6. Return to the mid-level and senior docs once you have changed real code.

## How to Use These Docs With Codex

Use Codex as a thinking partner, not as a replacement for your reasoning. A strong prompt is:

> I am studying `web/service/client.ts:68-90`. Ask me questions that help me explain how the oRPC client reaches the backend. Do not give me the answer first.

When implementing a story, ask for staged help:

> I am working on Story 3. I think the related pattern is in `web/service/knowledge/use-dataset.ts`, but I am unsure how query keys should change. Give me one hint and one file to inspect.

## How Not To Let AI Do All The Thinking

- Read the cited file range first.
- Write your own 3-5 sentence explanation.
- Ask Codex to critique the explanation.
- Only ask for implementation after you can name the files, data shape, permission boundary, and test path.
- Before accepting generated code, explain why every changed line belongs in the diff.

## 30-Day Self-Study Rhythm

- Days 1-5: read entry points, folder maps, simple components, and setup scripts.
- Days 6-10: trace dataset and app list flows from UI to database.
- Days 11-15: study React Query, oRPC contracts, request wrappers, auth wrappers, and Pydantic payloads.
- Days 16-20: trace workflow run/debug behavior and read the associated tests.
- Days 21-25: implement two easy user stories and ask Codex for review only after your first attempt.
- Days 26-30: perform a code review, write one missing test in a branch, and practice explaining Dify's architecture out loud.

