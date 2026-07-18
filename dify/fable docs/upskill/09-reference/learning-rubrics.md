# Learning Rubrics

Observable behaviors, not vibes. Grade yourself per skill; "Interview-ready" is the bar for claiming the skill in a mid-level loop.

| Skill | Junior (locate/explain) | Mid (change safely) | Senior (design/derisk) | Interview-ready when… |
| --- | --- | --- | --- | --- |
| Codebase navigation | Finds any feature's controller/service/component within 10 min using grep | Predicts which files a change touches before opening them | Maps ownership + contract strength of each surface | You can trace 2 of the 6 key flows at a whiteboard from memory |
| Backend layering | Explains the controller→service→core rule | Adds an endpoint keeping each layer honest | Detects and articulates boundary leaks (e.g. `is_agent`) with fixes | You can tell the layering-violation story with anchors in <2 min |
| Multi-tenancy | Knows `tenant_id` must scope queries | Writes cross-tenant rejection tests unprompted | Names non-DB tenancy surfaces (queues, caches, rate limits) | IDOR answer includes systematic prevention, not just definition |
| Async/queues | Explains why indexing is a Celery task | Ships a task with ids-only args + revalidation + idempotency | Designs fairness/outbox/sweeper tradeoffs with cost arguments | You can run the D1 debugging sim in ≤15 min cold |
| Streaming/SSE | Describes SSE vs WebSocket | Debugs a buffering issue hop-by-hop | Argues delivery-vs-execution decoupling + transport semantics | Flow-2 teach-back lands with a non-engineer listener |
| React state/data | useState vs store vs query classification | Ships a mutation with correct invalidations + tests | Audits a migration seam (dual keys) systemically | Q5/Q6 of 08/02 at mid bar, aloud |
| TypeScript | Avoids `any`; types responses | Narrows `unknown` at boundaries; catches drift risks | Designs codegen/contract strategy | You can explain where types *lie* (wire boundary) with the fetch.ts example |
| Testing | Runs targeted tests; reads neighboring tests first | Writes recipe-level tests (validation, permission, cross-tenant, dispatch) | Designs test strategy: what NOT to test and why | You defend a test-layer decision under pushback |
| Debugging | Reproduces reliably; reads error columns/logs | Bisects layers with cheap probes, narrating | Converts incidents into regression guards + alerts | Two timed sims passed with recording review |
| Security | Runs the checklist | Catches IDOR/XSS/SSRF in review katas | Articulates the three-principal LLM threat model | Kata 1 blockers found without hints |
| Performance | Names the domains | Measures before opining; finds an N+1 with SQL logging | Attaches budgets + regression guards | The apps-list N+1 story told with predicted-vs-measured numbers |
| Communication | Asks answerable questions | PRs follow the template; reviews are severity-ordered and kind | RFCs enable their own rejection; disagreements teach invariants | Stories 4 and 6 rehearsed under 2 min each |

## Self-assessment protocol

Monthly: pick five rows, grade honestly with *evidence* (link the PR, the recording, the drill artifact). No evidence = the lower grade. The gap list becomes next month's drill schedule.

## The mindset ladder, restated as behaviors

- Junior: ships when the happy path works.
- Mid: ships when the failure paths are handled and tested.
- Senior: ships when the *team* can operate, evolve, and roll back the thing — and the PR says how.
