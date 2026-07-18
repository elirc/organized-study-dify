# Dify Upskill Curriculum

A training lab built from this repository for one learner: a **junior fullstack JS engineer** (React/Node/TS CRUD background) who wants to reach mid-level fast, build senior judgment, and pass interviews for mid-level fullstack roles.

Every page teaches two things at once:

1. **This codebase** — where things live, how its real flows work, with exact file/line anchors.
2. **The transferable skill** — why the pattern exists, its failure modes, and how to talk about it in an interview.

## What this repo is

Dify is an open-source platform for building LLM applications: chatbots, agents, and node-graph "workflows," plus a RAG (retrieval-augmented generation) knowledge-base pipeline. It is a **polyglot monorepo**: a Python/Flask API (`api/`) organized in a layered controller → service → core/domain architecture, and a Next.js 16 + React 19 + TypeScript frontend (`web/`) using TanStack Query, Zustand, and ReactFlow. Async work runs through Celery with Redis; persistence is PostgreSQL (or MySQL) via SQLAlchemy plus a pluggable vector store (Weaviate et al.) for embeddings. Everything is **multi-tenant**: a `tenant_id` (workspace) must scope every query that touches shared tables. Model inference, streaming SSE responses, background indexing, and a plugin system make this one of the richest open codebases you can learn production architecture from.

Your stack familiarity (JS/TS) maps directly onto `web/`; the Python API is deliberately part of the curriculum — reading a second stack's layered backend is exactly what mid-level interviews probe ("can you learn an unfamiliar codebase?").

## Learning tracks

| Track | Folder | You'll be able to… |
| --- | --- | --- |
| Cartography | `01-codebase-cartography/` | Navigate any large repo deliberately |
| Stack mastery | `02-stack-and-language-mastery/` | Explain the runtime models you use daily |
| Architecture | `03-architecture-and-patterns/` | Recognize boundaries, critique designs |
| Reading gym | `04-code-reading-gym/` | Annotate, trace, and review real code |
| Quality | `05-quality-engineering/` | Test, debug, secure, and observe systems |
| Contribution | `06-contribution-practice/` | Ship PRs a maintainer would accept |
| Career | `07-career-and-collaboration/` | Review, write PRs/RFCs, communicate |
| **Interview prep** | `08-interview-prep/` | Answer with evidence, not trivia |
| Reference | `09-reference/` | Commands, risks, rubrics, verification |

## How to use it

- **One weekend**: [00-fast-track.md](00-fast-track.md) only.
- **Two weeks**: fast track → [01-.../05-key-flows.md](01-codebase-cartography/05-key-flows.md) → [03-.../05-pattern-catalog.md](03-architecture-and-patterns/05-pattern-catalog.md) → one ticket from [06-.../01-good-first-tickets.md](06-contribution-practice/01-good-first-tickets.md) → skim `08-interview-prep/`.
- **Eight weeks**: one track per week in numbered order, doing every drill; ship 2–3 tickets and one mid-level feature.
- **Ongoing contributor**: live in `06-contribution-practice/` and `07-career-and-collaboration/`, return to `03-…/06-architecture-critique.md` monthly.
- **Interview in two weeks**: go straight to [08-interview-prep/07-two-week-cram-plan.md](08-interview-prep/07-two-week-cram-plan.md).

### Recommended paths by profile

- **Brand-new junior**: 00 → 01 (all) → 02 → 04 (drills as you read) → 05 → 06 junior tickets.
- **Junior with JS/TS familiarity (you)**: 00 → 01/05-key-flows → 02 (skim JS, focus Python/Flask mental models) → 03 → 04 → 06.
- **Mid-level, new to the repo**: 01/01+05 → 03 → 05 → 06 mid tickets.
- **Senior doing architecture review**: 01/01-system-map → 03/06-architecture-critique → 09/risk-register.
- **Candidate, interview in two weeks**: 08/07-two-week-cram-plan (it schedules everything else).

## Conventions used throughout

- **File anchors**: `api/app_factory.py#L49-L124` or relative markdown links. Line numbers were confirmed against the working tree on 2026-07-10; if the code has moved, search for the named symbol.
- **Fake code**: any snippet not from this repo begins with `// Illustrative fake code: not from this repo` (or the `#` Python equivalent). Everything else is real.
- **Verification labels**: commands are marked __verified__ (actually run) or __inferred__ (read from scripts/config but not executed). See [09-reference/verification-log.md](09-reference/verification-log.md).
- **Drills and self-grading**: most sections end with a drill and a Basic/Solid/Strong rubric. Grade yourself honestly; "Strong" answers name invariants, risks, tests, and alternatives.
- **Interview angle**: sections flag where the material maps to a common interview question, with junior/mid/senior answer contrasts.

## The mindset ladder

- **Junior asks**: "How do I make it work?"
- **Mid-level asks**: "Is this the right pattern? What breaks it?"
- **Senior asks**: "What does this commit us to, who pays the cost, and how do we reduce the risk?"

Interviews for mid-level roles test exactly the second and third questions. A definition earns no points; a concrete example plus a tradeoff plus a failure mode does. This codebase gives you the concrete examples — the curriculum teaches you to wield them.
