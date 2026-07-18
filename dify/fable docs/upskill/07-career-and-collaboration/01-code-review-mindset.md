# Code Review Mindset

## The five layers (review in this order)

1. **Does it work?** — happy path correct; edge cases handled.
2. **Is it correct?** — invariants preserved: tenant scoping, status machines, transaction boundaries, idempotency.
3. **Will it stay correct?** — tests pin the behavior; types make misuse hard; failure paths observable.
4. **Does it fit?** — follows this repo's patterns (or deliberately improves them, stated in the PR); right layer; right surface (console vs service API).
5. **Is it kind to future maintainers?** — names, docstrings where non-obvious (per `api/AGENTS.md`), no cleverness debt.

Severity discipline: only layers 1–2 findings are **blocking**; layer 3 usually **important**; layers 4–5 mostly **optional** unless they hide a 1–2 issue. Mislabeling nits as blockers is the fastest way to lose review credibility.

## Repo-specific checklist

- [ ] Every new query with an external id is tenant/parent scoped (`api/AGENTS.md` rule; shape at `document_indexing_task.py#L82-L84`)
- [ ] Controller stays thin: Pydantic in, service call, error translation out (`completion.py` as reference shape)
- [ ] Auth decorator present and *ordered* (authn → init → resource → authz)
- [ ] Celery: ids not objects; re-validation inside the task; queue named explicitly; idempotent storage ops
- [ ] Commit before `.delay()`
- [ ] No transaction held across network calls (two-phase pattern, `document_indexing_task.py#L93-L120`)
- [ ] Sessions: context-managed with `expire_on_commit=False` where objects outlive the session
- [ ] Frontend: strings in i18n; overlays from `@langgenius/dify-ui/*`; mutations invalidate **both** key systems while the migration lasts
- [ ] New env config through `dify_config` + `.env.example`
- [ ] Errors: domain exceptions raised low, translated at the edge; log levels per house rule (warning=retryable)
- [ ] Public-surface changes (service API, DSL) are additive or versioned

## Comment craft

Formula: **observation → consequence → suggestion → severity**. Anchor to code, ask where unsure, and say what's good too.

> **[blocking]** This `select(Document).where(Document.id == doc_id)` has no dataset/tenant scope, so any authenticated user can fetch another workspace's document by id. Suggest matching the pattern in `document_indexing_task.py#L82-L84`. Happy to pair if the scoping source isn't obvious here.

> **[optional]** Nice use of the two-phase status pattern. Tiny naming thought: `process()` → `mark_parsing_and_run()` would make the transaction boundary visible at the call site.

Anti-patterns: "why didn't you just…" (adversarial), unexplained "LGTM" on complex diffs (rubber stamp), style nits on a correctness review (noise), rewriting the author's design in comments (that's an RFC conversation, not a review).

## Reviewing as a junior/mid (how to add value beyond seniors)

You can *always* contribute: trace the diff's happy path and confirm it; run it locally and report; check test coverage of the new branches; verify i18n/checklist items seniors skim past. Saying "I traced the indexing path with this change and the status transitions still hold; didn't evaluate the Redis protocol change" is a **great** review — scoped honesty beats false completeness.

Drill: review your own last PR (any repo) against the five layers, writing one finding per layer. Rubric — Strong: your layer-2 finding names an invariant by name (idempotency, scoping, ordering).
