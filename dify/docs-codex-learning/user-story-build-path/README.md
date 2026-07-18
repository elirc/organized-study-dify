# User Story Build Path

This suite gives you ten realistic feature plans to implement yourself. They are grounded in Dify's real app, dataset, workflow, API, auth, and testing patterns.

## How To Use These Stories

Pick one story, read every referenced file, write your own implementation checklist, then code on a branch. Do not ask Codex to implement the story first. Ask for hints, pattern explanations, or code review after your own attempt.

## Difficulty Progression

- Stories 1-3: small UI/validation changes touching one or two files.
- Stories 4-5: UI features that reuse existing state or data.
- Stories 6-7: new API/state behavior with validation or integration points.
- Stories 8-9: full-stack changes touching frontend, backend, and persistence.
- Story 10: architectural feature with security/operational implications.

## How To Know A Story Is Done

- Acceptance criteria are met.
- The behavior works manually.
- Tests are added or updated where the repo has an established pattern.
- The diff avoids unrelated refactors.
- You can explain the data flow and failure modes out loud.

## Codex Hint Prompt

"I'm working on Story X. I'm stuck on Y. Here is what I've tried: Z. Do not give me the solution immediately. Ask me questions that help me reason through the problem. Point me to the relevant files and patterns. Give hints in stages."

## Avoid Blind AI Acceptance

Before accepting AI-generated code, ask:

- Does this follow an existing file pattern?
- Did it touch unrelated files?
- Did it preserve tenant/auth boundaries?
- Did it update i18n strings when user-facing text changed?
- Did it include focused verification?

