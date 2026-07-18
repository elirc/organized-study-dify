# How To Use Codex With These Stories

## Start With Story 1

Read the files first. For Story 1, open `web/app/components/develop/tag.tsx:4-60` and `web/app/components/develop/__tests__/tag.spec.tsx:1-140`. Explain the behavior in your own words before asking for help.

## Ask For Hints Instead Of Full Solutions

Use:

> I am working on Story 2. I understand where `Datasets` renders cards, but I am unsure how to detect no results across infinite pages. Give me one hint and one file range to inspect.

## Ask Codex To Explain Existing Patterns First

Use:

> Explain how `web/service/knowledge/use-dataset.ts:83-99` builds a React Query key and URL. Do not write new code.

## Ask Codex To Review Your Attempt

Use:

> Review my diff for Story 6. Focus on query keys, backend validation, tenant scoping, and tests. Do not refactor unrelated code.

## Ask Codex To Create Tests After You Implement

Use:

> I implemented Story 3. Suggest focused tests using the repo's frontend test guide. Only cover the behavior I changed.

## Ask Codex To Explain A Diff

Use:

> Explain this diff as if I am preparing for PR review. Identify which layer each change belongs to and any missing verification.

## Prevent Unrelated Changes

Include:

> Do not modify production code outside the files required for this story. Do not reformat unrelated files. Preserve existing docs and tests unless directly relevant.

## Checklist Before Accepting AI Code

- It follows existing file patterns.
- It uses i18n for user-facing strings.
- It preserves tenant and auth checks.
- It includes focused tests or a clear verification reason.
- It does not add broad abstractions for a narrow story.
- You can explain every changed line.

## Checklist Before Opening A PR

- Acceptance criteria are checked.
- Focused tests pass or skipped reason is documented.
- Type/lint commands are run when feasible.
- Diff contains no unrelated changes.
- PR description names risks and verification.

## Example Prompts

Understanding a file:

> Walk me through `api/controllers/console/app/app.py:459-561` one block at a time. Ask me a question after each block.

Tracing a bug:

> Dataset search returns stale results. Help me trace from `web/app/components/datasets/list/index.tsx` to the backend. Give me the next file only after I summarize the current one.

Planning a change:

> For Story 8, help me produce a layer-by-layer implementation plan. Do not write code.

Reviewing a diff:

> Review my Story 7 diff for behavioral regressions. Prioritize bugs, missing tests, and side effects.

Writing tests:

> Based on my Story 1 implementation, propose Vitest cases using existing test style in `tag.spec.tsx`.

Hints only:

> I am stuck. Give me a hint, not the solution. Ask one question that helps me reason.

