# Security Checklist

Each row: the risk class, where this repo defends (or where to look), and what *you* check before merging.

| Risk | This repo | Pre-merge check |
| --- | --- | --- |
| **Authorization / IDOR** | Tenant scoping rule (`api/AGENTS.md`); resource loaders scoped (`get_app_model`); dataset-level ACLs (`DatasetPermission`, `dataset.py#L1355`) | Every query with a user-supplied id also filters `tenant_id`/parent id; every new route has an explicit authz decorator |
| **Authentication** | Console JWT + Redis refresh (`account_service.py#L200-L212`, `#L150-L159`); service-API tokens (`service_api/wraps.py#L55-L120`); passports for shares (`libs/passport.py`) | New endpoints declare which of the three surfaces they belong to; no "temporarily unauthenticated" routes |
| **CSRF** | Double-submit: cookie + header (`web/service/fetch.ts#L184`), verified in `login_required` (`libs/login.py#L99`) | Any cookie-authenticated mutation goes through the standard client (raw `fetch` bypasses the header) |
| **Input validation** | Pydantic at controllers (`completion.py#L43-L68`); service-level business checks | New payload fields validated (type + domain); no `request.json` raw reads in views |
| **SSRF** | Dedicated `ssrf_proxy` container (`docker-compose.middleware.yaml` L197); rule: outbound fetches via `core/helper/ssrf_proxy` (`api/AGENTS.md`) | Any feature fetching user-supplied URLs (website crawl, remote files, webhooks, tools) routes through the proxy helper — grep for naked `httpx/requests` usage in your diff |
| **XSS** | React escapes by default; markdown rendering via sanitizers (`dompurify` force-pinned in overrides — supply-chain floor) | No `dangerouslySetInnerHTML` with non-sanitized content (LLM output is untrusted!); check chat/markdown renderers when touching them |
| **Injection (SQL)** | SQLAlchemy expressions everywhere; "avoid raw SQL" (`api/AGENTS.md`) | No string-built SQL; LIKE patterns escaped (see `escape_query_for_search`, `hit_testing_service.py#L267`) |
| **Code execution** | User code runs in the **sandbox** container (compose L105), not in-process | Anything eval-like stays behind the sandbox service |
| **Secrets** | `dify_config` only; `.env` files git-ignored; SECRET_KEY signs all JWTs (`passport.py#L8-L9`) | No secrets in code/logs; new config via `configs/`, documented in `.env.example` (dotenv-linter runs in `make lint`) |
| **Dependency risk** | pnpm `overrides` force security floors; `minimumReleaseAge`, `trustPolicy`, `allowBuilds:false` (`pnpm-workspace.yaml`); uv lockfile | New deps: check maintenance, pin via catalog, justify in PR |
| **Uploads** | size/count limits (`BATCH_UPLOAD_LIMIT`, plan checks `dataset_service.py#L1982-L2006`); storage abstraction | Content-type validation, size caps, no path construction from filenames |
| **Webhooks/triggers** | `api/controllers/trigger/` — inbound surface | Signature verification, replay protection, and rate limits on any new inbound hook |
| **Cookies** | session + CSRF cookie pattern | `HttpOnly`/`SameSite`/`Secure` flags on any new cookie (verify in the login controller when touching it) |
| **Rate limiting** | app-level active-request limit (`app_generate_service.py#L114-L119`, `#L272-L291`); annotation import limits (`console/wraps.py#L340+`); token usage recording | New expensive endpoints get a limit decision *in the PR description*, even if the decision is "none, because…" |
| **Enumeration** | uniform login error "Invalid email or password" (`account_service.py#L219-L235`) | Error messages don't reveal existence of resources across tenant lines (404 vs 403 policy) |

## Threat model note specific to this product

LLM apps invert some instincts: **the prompt is user input, the output is untrusted, and the app author is not the platform**. Three distinct principals — platform operator, app builder (tenant), end user — each a threat to the others. Examples: an end user prompt-injecting an app into revealing its system prompt (builder's secret); a builder crafting a workflow that hammers outbound URLs (platform's SSRF problem — hence the proxy); the platform reading tenant data (mitigated by… policy; know the limits). Being able to lay out this three-principal model is a standout interview answer for "how do you think about security?"

## Pre-merge security checklist (copy into PRs)

- [ ] Every new query with external ids is tenant/parent-scoped
- [ ] New routes: authn surface named; authz decorator present; 403-vs-404 consistent
- [ ] All user-controlled URLs fetched via ssrf_proxy helper
- [ ] No unsanitized HTML sinks; LLM output treated as untrusted
- [ ] New config through `dify_config` + `.env.example`; no secrets committed
- [ ] Expensive endpoint? rate-limit decision recorded
- [ ] Errors don't leak internals or existence across tenants

Drill: take Kata 1 (conversation export) from [04-.../04-review-katas.md](../04-code-reading-gym/04-review-katas.md) and run this checklist against it line by line. Strong = you catch the two rows the kata's answer key emphasizes (IDOR, authz) *plus* the enumeration row (404 for other-tenant app ids, not 403).
