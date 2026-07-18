# 08 — Interview Prep

Target: **mid-level fullstack JS interviews**. Over 60% of the question cards here anchor to real Dify code so you practice answering with evidence instead of definitions.

## How mid-level loops are usually structured

1. **Screen** (30–45 min): background + one coding warmup or rapid-fire fundamentals.
2. **Technical deep-dive** (60 min): your projects; expect "why", "what broke", "what would you change".
3. **Practical coding** (60 min): build/extend a small feature; tests and edge cases matter more than speed.
4. **System design** (45–60 min): mid-level bar = coherent data model, API, and failure handling; senior bar adds capacity math and migration strategy.
5. **Behavioral** (45 min): STAR stories; they're listening for judgment and ownership.

## Using this repo as your portfolio

You studied (and ideally contributed to) a real multi-tenant LLM platform. That gives you legitimate first-person material: "In an open-source codebase I work in, chat streaming decouples execution into Celery with a replayable Redis event stream — here's why…" Every card below tells you which anchor to cite.

**The golden rule**: answer with *a concrete example + a tradeoff + a failure mode*. Never a bare definition. "What's idempotency?" → define in one sentence, then: "Dify's indexing task can rerun safely because it re-queries state and rebuilds segments; without that, retries would double chunks. The cost is re-doing work on retry."

| File | Contents |
| --- | --- |
| [01-js-ts-node-deep-dive.md](01-js-ts-node-deep-dive.md) | 14 cards: event loop, async, closures, TS |
| [02-frontend-framework-questions.md](02-frontend-framework-questions.md) | 12 cards: React 19, state, data fetching, perf |
| [03-api-and-data-modeling-questions.md](03-api-and-data-modeling-questions.md) | 12 cards: REST design, auth, schema, transactions |
| [04-system-design-from-this-repo.md](04-system-design-from-this-repo.md) | Full whiteboard walkthrough + variations |
| [05-debugging-and-code-review-rounds.md](05-debugging-and-code-review-rounds.md) | Timed simulations with rubrics |
| [06-behavioral-star-stories.md](06-behavioral-star-stories.md) | 10 STAR worksheets |
| [07-two-week-cram-plan.md](07-two-week-cram-plan.md) | Day-by-day schedule |
