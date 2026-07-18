# Behavioral STAR Story Worksheets

Ten worksheets. Stories 1–4 work even if you only *studied* the repo; 5–10 assume you shipped the referenced ticket/project from [06-contribution-practice](../06-contribution-practice/). Fill Situation/Task/Action/Result in your own words — the scaffolding here gives you the senior-signal details to include. Rehearsal check for every story: **under 2 minutes? concrete? ends with impact?**

---

## Story 1: "Walk me through learning a complex codebase"
Prompts: ramp-up speed, ambiguity, learning ability.
Source: this curriculum itself.
Scaffold: S — 500k+-line polyglot monorepo (Flask + Next.js), no team to ask. T — become productive fast. A — metadata-first method (AGENTS/Makefile), then traced one write path end-to-end (chat message: hook → SSE → decorators → service → Celery) before reading broadly; kept a verification log. R — could locate any feature's layers in minutes; found a real layering defect (`App.is_agent` committing inside a property getter).
Senior signals: *method*, not heroics; the defect find proves depth.
Resume bullet: "Reverse-engineered a 500k-line open-source LLM platform (Flask/Next.js); documented six end-to-end flows and surfaced a persistence-layer defect."

## Story 2: "Tell me about a technical tradeoff you analyzed"
Prompts: judgment, decision-making.
Source: commit-then-enqueue vs outbox (Flow 3, K3 kata).
Scaffold: A — quantified the crash window, weighed the outbox's moving parts against incident likelihood; recommendation depended on stuck-doc *data*, not doctrine. R — a written RFC-style analysis with a "do nothing (yet) + add sweeper" conclusion.
Senior signal: choosing the boring option with evidence.

## Story 3: "Describe a security issue you identified or prevented"
Source: tenant-scoping audit (T10 / security checklist work).
Scaffold: A — systematic grep for unscoped id lookups; wrote the cross-tenant test first; classified 404-vs-403 policy. R — pinned an isolation property with tests.
Senior signal: converting a one-off finding into a checklist + test pattern.

## Story 4: "Explain something complex to a non-expert" (communication probe)
Source: the delivery-vs-execution streaming architecture (Flow 2).
Scaffold: your 90-second plain-language version: "the answer is computed in a kitchen, the waiter just relays; if you step out, the kitchen keeps cooking and you replay the missed courses." R — used in a mock interview / written explainer.

## Story 5: "A bug you found and fixed" — from T12 (missing cache invalidation)
Scaffold: S — ghost rows after delete. A — bisected server-vs-cache in three probes; found the dual-key migration seam; fixed + hook test + audited siblings. R — class of bug closed, not one instance.
Maps to prompts: debugging, initiative, quality.

## Story 6: "You disagreed with a reviewer/maintainer" — from K4/kata 5 review exchange
Scaffold: A — the interests-not-positions reply about queue-boundary re-validation ([07/03](../07-career-and-collaboration/03-maintainer-communication.md)); conceded the code duplication point while defending the invariant; outcome = shared checker with both call sites.
Senior signal: the disagreement *taught the invariant* to the codebase via a comment.

## Story 7: "A time you improved reliability/operations" — from M3 (stale-indexing sweeper)
Scaffold: S — silent stuck documents. A — state-machine analysis → liveness gap → sweeper with dry-run + alert on queue age. R — stuck states became self-healing and visible.
Maps to: ownership, proactivity.

## Story 8: "Leading/executing a migration" — from M10/S3 (typed-client slice)
Scaffold: A — inventoried all readers before switching writers (the stale-cache lesson); resource-by-resource landing; deleted legacy per slice. R — zero regressions, burndown visible.
Senior signal: sequencing and blast-radius control, not code volume.

## Story 9: "Handling ambiguity / underspecified work" — from M7 (are api_rpm/rph enforced?)
Scaffold: A — archaeology: traced the columns (`model.py#L411-L412`) to (possibly) nothing; turned "add headers" into "first establish what's true," reported findings before building. R — the honest scope change.
Maps to: ambiguity, intellectual honesty. Interviewers love this one.

## Story 10: "A mistake you made" (pick the real one when it happens; template)
Scaffold requirement: real, small-blast-radius, caught-by-you, with a *system* fix (test/checklist/alert), not "I'll be more careful."
Example shape if drawn from this work: shipped the T13 empty-query validator without grepping suggested-questions callers; a feature broke in preview; added the grep step to your personal pre-PR checklist. R — the checklist caught two later issues.

---

## Prompt → story map

| Common prompt | Stories |
| --- | --- |
| Complex codebase / ramp-up | 1 |
| Tradeoff / architecture decision | 2, 8 |
| Conflict / disagreement | 6 |
| Bug war story | 5, 7 |
| Ambiguity | 9 |
| Mistake / failure | 10 |
| Security awareness | 3 |
| Communication | 4, 6 |
| Initiative / ownership | 5, 7, 9 |

Rehearse the top row of each prompt cluster; record; cut every sentence that doesn't carry either evidence or impact.
