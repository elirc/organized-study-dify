# Debugging and Code Review Rounds — Timed Simulations

Run these with a timer, aloud, ideally with a friend playing interviewer reading the follow-ups. Grading rubric at the bottom applies to all.

---

## Debugging sim D1 (20 min): "Documents stuck in processing" (from [05/03 scenario 1](../05-quality-engineering/03-systematic-debugging.md))

Interviewer setup: "Users report knowledge documents stuck at 'parsing' for hours. You have DB access, Redis access, and logs. Go."
Expected narration: bisect dispatch-vs-execution using `processing_started_at` (`document_indexing_task.py#L104`); then worker liveness, tenant-queue flag state, error columns.
Follow-ups the interviewer fires: "The timestamp is set but no logs from the runner — now what?" (worker OOM/kill mid-run) → "How do you prevent recurrence?" (sweeper + liveness alert) → "Write the sweeper's WHERE clause."
Pass: reaches a *class* of root cause in ≤10 min with cheap discriminating probes; proposes regression coverage unprompted.

## Debugging sim D2 (15 min): "Deleted app still in the list" (scenario 3)

Setup: "No errors anywhere. DELETE returns 200."
Expected: client-cache hypothesis within 2 minutes (server 200 + UI stale = cache); Query devtools; dual-key migration explanation (`use-apps.ts#L70-L78`).
Follow-ups: "Why do two key systems exist?" (migration) → "How do you find every other mutation with this bug?" (grep audit + centralized key factory as the fix-forward).
Pass: correct layer in ≤3 probes; systemic fix proposed, not just the one-line patch.

## Debugging sim D3 (20 min): "Workflow spinner forever" (scenario 5)

Setup: "One tenant's workflow runs never finish in the UI; backend says runs succeeded."
Expected: compute-vs-delivery split stated *first*; then transport semantics (pub/sub lost events vs streams replay), subscriber lifecycle, `retrieve_events` re-subscribe.
Follow-ups: "Only this tenant — why?" (their proxy kills idle SSE; or their runs finish before subscribe on pub/sub) → "Design the fix." (streams transport / replay-from-zero).
Pass: the compute/delivery sentence in the first minute — it's the whole sim.

## Review sim R1 (25 min): conversation-export PR ([04/04 Kata 1](../04-code-reading-gym/04-review-katas.md))

Format: read the fake diff summary, deliver a spoken review in severity order.
Interviewer follow-ups: "The author says tenant scoping is handled by login. Respond." (authn ≠ authz ≠ resource scoping — three different checks) → "They push back on streaming as premature. Where's your line?" (bounded memory is correctness, not optimization, at export scale).
Pass: both blockers found; disagreement handled with the interests-not-positions pattern ([07/03](../07-career-and-collaboration/03-maintainer-communication.md)).

## Review sim R2 (20 min): retry-LLM-calls PR (Kata 3)

Follow-ups: "Which errors *should* retry?" (transient provider 5xx/timeouts; never quota/auth) → "Where does retry belong architecturally?" (a layer that sees typed errors; never sleeping in a request worker) → "The author cites a 30% failure rate — does that change anything?" (yes: that's an *availability incident*, not a retry-tuning problem; investigate the provider).
Pass: severity calibration + the last follow-up trap avoided (don't tune around a fire).

## Review sim R3 (15 min): module-scope cache PR (Kata 2)

Follow-ups: "We run one worker in dev and it works." (multi-worker prod + no invalidation) → "Redis then?" (yes, with TTL + invalidation on write; or push caching to the client layer that already exists).
Pass: names *all three* defects of module-scope caching (staleness, per-process divergence, memory growth) within the first two minutes.

---

## Universal grading rubric

| Dimension | Weak | Strong |
| --- | --- | --- |
| Structure | jumps at hypotheses | states the bisection *before* probing |
| Probes | expensive first (read all logs) | cheapest discriminating probe first |
| Narration | silent thinking | continuous "what I expect to see if X" |
| Root cause | stops at symptom patch | class of cause + regression guard |
| Review tone | commands / nitpicks | severity-ordered, kind, evidence-anchored |
| Follow-up grace | defensive | updates position visibly when given new info |

Record yourself once per sim. The recording is brutal and worth it.
