# 30-Day Technology Practice Plan

## Day 1

**Focus Area:** Repo map  
**Files to Read:** `package.json`, `web/package.json`, `api/pyproject.toml`  
**Concept:** A monorepo is a set of related systems with separate commands and responsibilities.  
**Small Exercise:** Write down which scripts are frontend, backend, E2E, and root-level.  
**Self-Check Question:** Why can running a command from the wrong directory fail even when the command exists?  
**Optional Codex Prompt:** "Ask me questions that help me classify this repo's package scripts. Do not summarize them for me yet."

## Day 2

**Focus Area:** Setup workflow  
**Files to Read:** `Makefile:17-46`, `api/.env.example`, `web/.env.example`  
**Concept:** Setup scripts encode assumptions about middleware, env files, and migrations.  
**Small Exercise:** Reconstruct the local setup sequence in your own words.  
**Self-Check Question:** Which steps prepare files, which install dependencies, and which touch services?  
**Optional Codex Prompt:** "Quiz me on this Makefile setup flow one step at a time."

## Day 3

**Focus Area:** Frontend root  
**Files to Read:** `web/app/layout.tsx:29-91`  
**Concept:** Root layouts wire app-wide dependencies.  
**Small Exercise:** Draw the provider stack and explain what breaks if React Query is missing.  
**Self-Check Question:** Which providers are UI infrastructure and which are data/state infrastructure?  
**Optional Codex Prompt:** "Give me hints to explain this provider stack without giving me the answer."

## Day 4

**Focus Area:** App context  
**Files to Read:** `web/context/app-context-provider.tsx:30-159`  
**Concept:** Context should expose stable app-wide facts, not every feature's state.  
**Small Exercise:** List every value exposed by the provider and classify it.  
**Self-Check Question:** Why do role booleans belong near app context?  
**Optional Codex Prompt:** "Ask me to identify the boundaries of AppContext."

## Day 5

**Focus Area:** Simple React component  
**Files to Read:** `web/app/components/develop/tag.tsx:4-60`, `web/app/components/develop/__tests__/tag.spec.tsx:1-140`  
**Concept:** A reusable component is a small contract.  
**Small Exercise:** Explain each prop and one behavior it controls.  
**Self-Check Question:** What would make this component too feature-specific?  
**Optional Codex Prompt:** "Give me staged hints to understand this component's prop API."

## Day 6

**Focus Area:** TypeScript props  
**Files to Read:** `web/app/components/datasets/list/datasets.tsx:11-38`  
**Concept:** Props document how parent and child components cooperate.  
**Small Exercise:** Trace each prop to where it is used.  
**Self-Check Question:** Which props are display data and which are behavior callbacks?  
**Optional Codex Prompt:** "Question me on this component's prop contract."

## Day 7

**Focus Area:** Dataset screen  
**Files to Read:** `web/app/components/datasets/list/index.tsx:25-108`  
**Concept:** Feature components combine local UI state with reusable data hooks.  
**Small Exercise:** Trace search text from local state into the data hook.  
**Self-Check Question:** Why does debounce matter for search?  
**Optional Codex Prompt:** "Help me reason through this dataset list flow without solving it for me."

## Day 8

**Focus Area:** React Query  
**Files to Read:** `web/context/query-client.tsx:1-28`, `web/service/knowledge/use-dataset.ts:28-104`  
**Concept:** Server state needs cache identity.  
**Small Exercise:** Identify the dataset list query key inputs.  
**Self-Check Question:** What bug appears if the query key omits a filter?  
**Optional Codex Prompt:** "Ask me to debug a hypothetical stale dataset list."

## Day 9

**Focus Area:** HTTP wrapper  
**Files to Read:** `web/service/fetch.ts:137-249`  
**Concept:** Shared transport code prevents every feature from reinventing auth, CSRF, and response parsing.  
**Small Exercise:** Trace how a JSON body becomes a request.  
**Self-Check Question:** Where is the CSRF header added?  
**Optional Codex Prompt:** "Quiz me on this request wrapper's responsibilities."

## Day 10

**Focus Area:** Type discipline  
**Files to Read:** `web/service/fetch.ts:21-24`, `web/service/base.ts:41-132`  
**Concept:** Types are strongest when they describe boundary behavior precisely.  
**Small Exercise:** Find one strong type and one weak type.  
**Self-Check Question:** Why is `any` riskier in shared infrastructure than in a narrow test mock?  
**Optional Codex Prompt:** "Give me a hint-only critique of these TypeScript types."

## Day 11

**Focus Area:** API contracts  
**Files to Read:** `web/contract/router.ts:56-120`, `web/contract/console/apps.ts:6-47`  
**Concept:** Contracts make API shape visible to the frontend.  
**Small Exercise:** List the app contracts and their route paths.  
**Self-Check Question:** What does the incomplete generated-contract note imply for reviewers?  
**Optional Codex Prompt:** "Ask me to compare manual and generated contract risks."

## Day 12

**Focus Area:** Backend entry  
**Files to Read:** `api/app.py:32-62`, `api/app_factory.py:49-138`  
**Concept:** Backend apps should make initialization explicit.  
**Small Exercise:** Trace normal app startup from `app.py` into the factory.  
**Self-Check Question:** Why does the migration app path exist separately?  
**Optional Codex Prompt:** "Question me on the Flask app factory."

## Day 13

**Focus Area:** Backend extensions  
**Files to Read:** `api/app_factory.py:177-219`, `api/extensions/ext_blueprints.py:25-100`  
**Concept:** Extensions and blueprints define backend infrastructure and route families.  
**Small Exercise:** Name each blueprint family and what it likely owns.  
**Self-Check Question:** Why is route grouping important in a large API?  
**Optional Codex Prompt:** "Give me hints to map the blueprint families."

## Day 14

**Focus Area:** Backend validation  
**Files to Read:** `api/controllers/console/app/app.py:66-131`  
**Concept:** Pydantic turns raw HTTP input into safe application input.  
**Small Exercise:** Explain the tag ID validator.  
**Self-Check Question:** What bad inputs does this model prevent?  
**Optional Codex Prompt:** "Quiz me on Pydantic boundary validation."

## Day 15

**Focus Area:** Backend route anatomy  
**Files to Read:** `api/controllers/console/app/app.py:459-561`  
**Concept:** A route method is a boundary, not the whole feature.  
**Small Exercise:** Annotate decorators, parser, validation, service call, and response.  
**Self-Check Question:** Which line hands off to business logic?  
**Optional Codex Prompt:** "Ask me to identify every responsibility in this route."

## Day 16

**Focus Area:** Service layer  
**Files to Read:** `api/services/app_service.py:56-203`  
**Concept:** Services coordinate business decisions and persistence.  
**Small Exercise:** Trace app creation from validation to commit.  
**Self-Check Question:** Why should this logic not live entirely in the controller?  
**Optional Codex Prompt:** "Give me staged hints on the app creation flow."

## Day 17

**Focus Area:** SQLAlchemy model basics  
**Files to Read:** `api/models/base.py:14-104`, `api/models/model.py:392-490`  
**Concept:** Models define persistent shape and relationships.  
**Small Exercise:** List key `App` fields and one property that may query.  
**Self-Check Question:** What is the risk of query-backed properties in list endpoints?  
**Optional Codex Prompt:** "Ask me performance questions about this model."

## Day 18

**Focus Area:** Dataset backend  
**Files to Read:** `api/controllers/console/datasets/datasets.py:130-207`, `api/controllers/console/datasets/datasets.py:289-450`, `api/models/dataset.py:166-220`  
**Concept:** A domain feature crosses validation, route, and model layers.  
**Small Exercise:** Trace dataset list request fields to model fields.  
**Self-Check Question:** Which layer owns `enable_api` persistence?  
**Optional Codex Prompt:** "Help me trace dataset API behavior by asking questions."

## Day 19

**Focus Area:** Auth and tenant boundaries  
**Files to Read:** `api/libs/login.py:38-118`, `api/controllers/console/app/wraps.py:13-145`  
**Concept:** Security checks are often decorators and wrappers, not only code inside methods.  
**Small Exercise:** Explain how current account and app tenant loading work.  
**Self-Check Question:** What could go wrong if a route loads an app without tenant scope?  
**Optional Codex Prompt:** "Quiz me on auth and tenant safety."

## Day 20

**Focus Area:** Workflow editor shell  
**Files to Read:** `web/app/components/workflow-app/index.tsx:40-155`  
**Concept:** Complex UI shells compose providers, stores, hooks, and effects.  
**Small Exercise:** Identify which logic belongs to setup rather than rendering.  
**Self-Check Question:** Why would moving run logic directly into this component be risky?  
**Optional Codex Prompt:** "Ask me to separate responsibilities in this component."

## Day 21

**Focus Area:** Zustand workflow state  
**Files to Read:** `web/app/components/workflow-app/store/workflow/workflow-slice.ts:3-35`  
**Concept:** Local editor state has different rules than server state.  
**Small Exercise:** Classify each field in the slice.  
**Self-Check Question:** Which fields would be dangerous to store only on the server?  
**Optional Codex Prompt:** "Give me hint-only feedback on my state classification."

## Day 22

**Focus Area:** Workflow run hook  
**Files to Read:** `web/app/components/workflow-app/hooks/use-workflow-run.ts:99-516`  
**Concept:** Side-effect hooks orchestrate multi-step user actions.  
**Small Exercise:** Draw the sequence inside `handleRun`.  
**Self-Check Question:** Why does draft synchronization happen before execution?  
**Optional Codex Prompt:** "Ask me to trace handleRun chronologically."

## Day 23

**Focus Area:** Workflow backend route  
**Files to Read:** `api/controllers/console/app/workflow.py:251-367`, `api/controllers/console/app/workflow.py:697-759`  
**Concept:** Workflow persistence and run lifecycle are separate backend operations.  
**Small Exercise:** Compare draft sync, draft run, and task stop endpoints.  
**Self-Check Question:** What user-visible bug appears if stop does not match the running task?  
**Optional Codex Prompt:** "Quiz me on workflow route responsibilities."

## Day 24

**Focus Area:** Workflow service  
**Files to Read:** `api/services/workflow_service.py:223-318`  
**Concept:** Service methods enforce domain constraints before persistence.  
**Small Exercise:** List validations before draft workflow save.  
**Self-Check Question:** Why should graph validation happen server-side too?  
**Optional Codex Prompt:** "Ask me to explain server-side workflow validation."

## Day 25

**Focus Area:** Frontend testing  
**Files to Read:** `web/app/components/develop/__tests__/tag.spec.tsx:1-140`  
**Concept:** Test behavior users and callers depend on.  
**Small Exercise:** Add a hypothetical test case in notes, not code.  
**Self-Check Question:** Which assertion tests behavior rather than implementation?  
**Optional Codex Prompt:** "Review my proposed test idea without writing the test."

## Day 26

**Focus Area:** Hook testing  
**Files to Read:** `web/app/components/workflow-app/hooks/__tests__/use-workflow-run.spec.ts:1-180`  
**Concept:** Mock dependencies to isolate orchestration logic.  
**Small Exercise:** List all mocked dependencies and why each is mocked.  
**Self-Check Question:** What would make this test too brittle?  
**Optional Codex Prompt:** "Ask me to identify test doubles in this hook test."

## Day 27

**Focus Area:** Backend testing  
**Files to Read:** `api/tests/unit_tests/tasks/test_workflow_execute_task.py:73-160`  
**Concept:** Backend unit tests should protect behavior without requiring full external systems.  
**Small Exercise:** Identify arrange, act, and assert sections.  
**Self-Check Question:** What behavior is the test protecting?  
**Optional Codex Prompt:** "Quiz me on the AAA structure in this pytest test."

## Day 28

**Focus Area:** Code review practice  
**Files to Read:** `web/service/fetch.ts:21-24`, `api/models/dataset.py:213-220`, `web/contract/router.ts:65-69`  
**Concept:** Reviewers look for risk, not just style.  
**Small Exercise:** Write three review comments: one type-safety, one performance, one contract-drift.  
**Self-Check Question:** Did each comment explain user impact or maintenance risk?  
**Optional Codex Prompt:** "Grade my review comments for specificity and fairness."

## Day 29

**Focus Area:** Feature planning  
**Files to Read:** `web/service/knowledge/use-dataset.ts:158-187`, `api/controllers/console/datasets/datasets.py:381-450`, `api/models/dataset.py:166-220`  
**Concept:** Good implementation plans name every layer before coding.  
**Small Exercise:** Plan a small dataset setting change without writing code.  
**Self-Check Question:** Did you include UI, hook, API, validation, model, and tests?  
**Optional Codex Prompt:** "Ask me questions to improve this feature plan without solving it."

## Day 30

**Focus Area:** Explain the system  
**Files to Read:** `web/app/layout.tsx:61-84`, `web/service/knowledge/use-dataset.ts:28-104`, `api/app_factory.py:49-138`, `api/controllers/console/app/app.py:459-561`, `api/services/app_service.py:56-203`, `api/models/model.py:392-490`  
**Concept:** Senior engineers explain systems through flows and trade-offs.  
**Small Exercise:** Give a five-minute walkthrough of Dify's frontend-to-backend feature flow.  
**Self-Check Question:** Did you mention boundaries, contracts, validation, persistence, testing, and risks?  
**Optional Codex Prompt:** "Interview me on this repo. Ask follow-up questions, but do not answer for me."

