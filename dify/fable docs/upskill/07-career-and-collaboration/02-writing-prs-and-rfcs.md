# Writing PRs and RFCs

## PR description template (use for every ticket in module 06)

```markdown
## What
One paragraph: the user-visible or maintainer-visible change.

## Why
The problem/ticket; link issues. If reviving an old discussion, link it.

## How
Bullet the non-obvious decisions, each with its alternative:
- Chose commit-then-enqueue over outbox because … (blast radius, existing pattern at dataset_service.py#L2269-L2277)

## How tested
- `make test TARGET_TESTS=…` output summary
- Manual: steps a reviewer can replay in <5 minutes

## Risks
What could break, who would notice, how we'd know (log/metric/status).

## Follow-ups
Deliberately deferred work, so reviewers don't re-litigate scope.
```

Rules: PRs < ~400 lines of substantive diff (split otherwise — mechanical moves separate from behavior changes, the K2 lesson); screenshots for UI; the *first comment on your own PR* pre-empting the hardest review question buys enormous goodwill.

## Commit messages

`type(scope): imperative summary` matches this repo's ecosystem norms — e.g. `fix(api): scope document lookup by dataset_id`. Body answers *why*; reference the invariant when relevant ("preserves tenant isolation rule from api/AGENTS.md").

## When to RFC instead of PR

RFC when any of: touches a public contract (service API, DSL), adds infrastructure (new table used by many, new queue, new daemon), crosses team ownership, or the *rollback is hard* (data deletion, migrations on huge tables). One-way doors get documents; two-way doors get PRs. (S1/S4/S5 in [06/03](../06-contribution-practice/03-senior-build-projects.md) are RFC-first; every M-ticket is PR-first with a design note.)

## RFC template tailored to this repo

```markdown
# RFC: <name>
Status: draft | review | accepted | rejected
## Problem
Evidence-first: anchors, incident counts, measurements. (E.g., stuck-PARSING rate.)
## Non-goals
## Proposal
Data model delta (migration sketch) · API delta (which surface: console/service/web)
· Task/queue changes · Config (dify_config additions) · Multi-tenancy statement
  ("every new query is scoped by …") · Failure modes + observability plan
## Alternatives (including Do Nothing, honestly costed)
## Migration & rollout
Order of PRs; feature flags; backfill plan; rollback per stage.
## Open questions
```

The multi-tenancy statement and the surface declaration are the two repo-specific sections — reviewers here will ask about both every time; answer preemptively.

## Drill

Write the full PR description for ticket T10 (404/403 consistency) *before* implementing it. Rubric — Strong: your Risks section names the client-behavior change (web app code that branches on 403 vs 404 — you grepped for it) and your test list covers both tenants and both statuses.
