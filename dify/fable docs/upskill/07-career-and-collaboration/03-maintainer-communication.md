# Maintainer Communication

Open source (and every job) runs on asynchronous trust. The skill: **bring evidence, show your work, make responding cheap.**

## Asking for help without outsourcing thinking

Bad: "The indexing doesn't work, any ideas?"
Good structure — context, what you tried, where you're stuck, specific question:

> I'm implementing the stale-indexing sweeper (issue #X). Documents stuck in `parsing` are selected by `processing_started_at < now - N` (`document_indexing_task.py#L104` sets it). My question: documents paused via `DocumentIsPausedError` also sit non-terminal — is there a flag distinguishing paused from crashed, or should the sweeper treat pause as legitimate and skip it? I've checked `models/dataset.py` `Document` columns and found `is_paused`-like fields but want to confirm intended semantics.

Answerable in one sentence, shows an hour of homework, includes anchors. That's the bar.

## Bug reports / repro reports

Template: **Expected / Actual / Minimal repro / Environment / Evidence / Suspicion (labeled)**.

> **Expected**: deleting an app removes it from the list immediately.
> **Actual**: ghost row until hard refresh.
> **Repro**: create 2 apps → delete one from the grid menu → observe list. 100% on `main`, Chrome.
> **Evidence**: DELETE returns 200; React Query devtools show `['apps','full-list']` invalidated but screen reads `consoleQuery.apps.list` key. (Screens recording attached.)
> **Suspicion** (unverified): missed dual-key invalidation during the client migration — cf. the pattern in `web/service/use-apps.ts#L70-L78`.

Minimal examples: strip to the smallest thing that still fails. For API bugs, a `curl` line; for UI, a 15-second recording beats three paragraphs.

## Proposing a feature

Lead with the problem, not the solution; size it honestly; volunteer the work: "Problem: operators can't see stuck documents (evidence…). Proposal sketch: … . Happy to write the RFC and implement if maintainers agree the direction fits — is there prior art or a reason it's been avoided?" The last clause matters: it invites the "we tried that" history you can't see.

## Respectful disagreement

Rules: argue interests (correctness, blast radius, maintenance) not positions ("my way"); ask for the constraint you might be missing; concede genuinely when convinced; escalate *scope*, not volume (a thread going nowhere → suggest a call or an RFC, don't repeat louder).

> I might be missing context — is the duplicate billing check in the task intentional as a queue-boundary defense? If so I'll add a comment documenting it instead of removing it (my original proposal). If it's accidental, I'd still keep one check inside the task since plan state can change between enqueue and run.

Notice: the disagreement itself teaches the reader the invariant. That's the tone.

## Responding to review on your PR

- Every comment gets a response: fix, push, and say "done — <commit>"; or a reasoned "kept as-is because…, happy to change if you feel strongly."
- Batch pushes; don't force-push over a reviewer's in-progress read mid-review.
- Thank people for catches that saved you — specifically, not ritually.

Drill: take the M3 sweeper ticket and write (1) the help-request you'd post about pause semantics, and (2) the disagreement reply if a maintainer says "just increase the Celery timeout instead." Rubric — Strong: your disagreement quantifies why timeout ≠ liveness (worker death, not slowness, is the failure mode) and still leaves the maintainer a graceful path to be right.
