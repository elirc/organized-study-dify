# 03 — Architecture and Patterns

This module turns "I can read the code" into "I can judge the code." Vocabulary you'll use throughout (and in interviews — these are interview words):

- **Boundary**: a line across which only declared shapes pass (HTTP payloads, task args).
- **Invariant**: something that must always hold (every query is tenant-scoped).
- **Contract**: a shape/behavior others rely on without reading your code.
- **Ownership**: which layer is allowed to change a piece of state.
- **Idempotency**: safe to run twice.
- **Isolation**: one tenant/user/request can't observe or corrupt another.
- **Blast radius**: how much breaks if this change is wrong.

| File | Covers |
| --- | --- |
| [01-boundaries-and-layers.md](01-boundaries-and-layers.md) | Controller → service → core layering; good boundaries and real leaks |
| [02-data-model-and-persistence.md](02-data-model-and-persistence.md) | Schema, sessions, transactions, migrations |
| [03-validation-auth-and-permissions.md](03-validation-auth-and-permissions.md) | Every validation layer; authn vs authz; tenant isolation |
| [04-side-effects-async-and-reliability.md](04-side-effects-async-and-reliability.md) | Celery, queues, retries, idempotency, event delivery |
| [05-pattern-catalog.md](05-pattern-catalog.md) | 14 pattern cards — recognition training |
| [06-architecture-critique.md](06-architecture-critique.md) | Strengths, risks, and what to change — doubles as system-design prep |
