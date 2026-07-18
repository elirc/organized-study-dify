# JS / TS / Node Deep-Dive Cards

Format per card: what it's testing · repo anchor · junior/mid/senior answer contrast · follow-ups · drill. Practice aloud, 90 seconds per answer.

---

## Q1: Explain the event loop. What are microtasks vs macrotasks?
Testing: runtime model fundamentals.
Anchor: SSE consumption in `web/service/base.ts#L452-L580` — incremental reads keep the loop free.
Junior: one thread, callback queue, promises run "later."
Mid: microtasks (promise reactions) drain fully between macrotasks; starvation possible; concrete example: processing SSE chunks via promise chains never blocks rendering because each read yields back to the loop.
Senior: adds scheduling interactions (rAF vs microtask for UI batching — relevant to chat re-render pressure) and when to *deliberately* defer work.
Follow-ups: "What does `await` compile to?" "Why can a tight promise loop freeze the UI?"
Drill: whiteboard the ordering of `setTimeout(0)` vs `Promise.resolve().then` vs sync code.

## Q2: How does streaming a fetch response work? Why not just await `res.json()`?
Testing: streams, backpressure awareness.
Anchor: `handleStream` reading the body reader (`base.ts#L189+`, invoked `#L555`).
Junior: "you read chunks."
Mid: `res.body.getReader()`, decode incrementally, parse SSE frames, dispatch typed callbacks; `res.json()` would buffer the entire answer — no first-token UX.
Senior: delivery vs execution decoupling on the server side (Celery + replay), reconnect semantics, and why SSE over WebSocket for one-directional streams.
Follow-ups: partial frame across chunk boundary — how do you buffer?
Drill: write the SSE line-splitting loop from memory.

## Q3: What's a closure? Show a closure bug you've seen.
Testing: scoping fundamentals via war story.
Anchor: `isRespondingRef` (`chat/hooks.ts#L637-L640`) and the 10-dep `useCallback` (`#L1170-L1181`).
Junior: function remembering outer variables.
Mid: stale closure in long-lived callbacks: a `useCallback` capturing old state; fix = ref-for-latest or correct deps; cite the chat guard.
Senior: taxonomy — when you *want* capture (module pattern: the `_try_start` lock closure pattern, though that one's Python — the JS analogue is the single-flight refresh) vs when you want references.
Follow-ups: "why do refs escape re-render capture?"

## Q4: How do you cancel async work in JS?
Testing: resource lifecycle discipline.
Anchor: `AbortController` handed out via `getAbortController` (`base.ts#L493-L513`); previous-subscription abort (`chat/hooks.ts#L1158-L1160`).
Junior: "AbortController exists."
Mid: create per operation; owner aborts on supersede/unmount; fetch rejects with AbortError which you must filter from real errors (the string-matching at `base.ts#L561` — and why that's fragile).
Senior: cancellation as an API-design concern — who owns the controller, propagation through layers, and idempotent server semantics for cancelled writes (the stop-task endpoint `completion.py#L203-L224` is the server half).
Follow-ups: cancel a `useEffect` fetch; cancel React Query queries.

## Q5: Describe a race condition and how you'd prevent it.
Testing: concurrency reasoning.
Anchor: subscribe-gated task start — lock + fallback timer (`app_generate_service.py#L41-L85`); JS analogue: concurrent token refresh (`refresh-token.ts` — check its single-flight).
Mid: two triggers, one action → guard flag with atomic check-and-set; in JS single-thread the interleaving comes from await points.
Senior: names the transport reason the race exists at all (at-most-once pub/sub) — fixing the *requirement* beats fixing the race.
Drill: write the double-submit-guard for a form in both flag and promise-caching styles.

## Q6: `unknown` vs `any`? When is casting acceptable?
Anchor: `const data: unknown = await response.clone().json()` then narrow (`fetch.ts#L66-L67`).
Mid: `unknown` forces narrowing; `any` disables checking and *spreads*; boundaries (network, storage) parse to unknown then validate.
Senior: cost model — `as` is a promise you make the compiler; acceptable when an invariant is enforced elsewhere and documented; repo bans `any` outright (`AGENTS.md`).
Follow-ups: `satisfies` vs `as`; exhaustiveness with `never`.

## Q7: How do generics actually help in application code (not libraries)?
Anchor: `base<T>(url): Promise<T>` (`fetch.ts#L137`), `useQuery<App>` (`use-apps.ts#L44-L46`).
Mid: typed HTTP layer — one implementation, per-call response contracts; inference flows to component props.
Senior: where this *lies* (T is asserted, not validated — wire drift) and the codegen/zod remedies; that honesty is the differentiator.

## Q8: Discriminated unions — where would you use them?
Anchor: the ~30 SSE event types dispatched to typed callbacks (`base.ts#L57-L132`).
Mid: `{event: 'node_started', data: …} | {event: 'message_end', …}` with a switch and exhaustive `never` check.
Senior: protocol evolution — adding a variant must fail compilation at every consumer; contrast with the callback-bag approach the repo uses and its tradeoff (unknown events are silently ignored = forward-compatible but silently lossy).

## Q9: How do promises interact with error handling? Where do errors get lost?
Anchor: `ssePost`'s `.then` chain with internal `.json().then(...)` branches (`base.ts#L521-L553`).
Mid: unawaited promise rejections vanish (until unhandledrejection); mixing then/await styles hides paths; always terminate chains.
Senior: error *taxonomy* design — the repo converts HTTPError to synthetic Response (`fetch.ts#L232-L235`) so callers handle one shape; the cost is surprising `throw`n Responses.

## Q10: Node vs browser: what changes in your code?
Anchor: `web/service/fetch.ts` uses `globalThis.fetch`, `js-cookie`, `location` — browser-bound; Next server components / `proxy.ts` run in Node.
Mid: different globals, no DOM/cookies server-side, env secrets only on server; the repo marks client-only deps (`client-only` package in catalog).
Senior: the SSR boundary as a security boundary (what leaks into the client bundle) and the `@t3-oss/env-nextjs` pattern (`web/env.ts`) for split env validation.

## Q11: How does module resolution/bundling affect app structure?
Anchor: `eslint-plugin-no-barrel-files` in root devDeps; pnpm catalog overrides aliasing vite → vite-plus.
Mid: barrels create import cycles and kill tree-shaking; path aliases (`@/`) need tsconfig+bundler agreement.
Senior: supply-chain angle — overrides/pinning as part of module strategy (`pnpm-workspace.yaml` overrides, `minimumReleaseAge`).

## Q12: Explain `async` iteration / generators. Where do they shine?
Anchor: the Python twins are everywhere (`WorkflowEntry.run` yields events, `workflow_entry.py#L222-L234`); JS: the SSE reader loop is a hand-rolled async iterator.
Mid: `for await (const chunk of stream)` for backpressure-respecting consumption.
Senior: generators as *protocol adapters* — converting engine events to HTTP frames (`convert_to_event_stream` conceptually) — the same shape in both languages; being able to say that cross-language sentence is senior signal.

## Q13: What happens between typing a URL and seeing the page (frontend-weighted)?
Testing: breadth check.
Anchor: use Dify's console boot: `web/app/layout.tsx` → providers → `AppContextProvider` fetching `/console/api/workspaces/current` (exempt-list comment `app_factory.py#L20-L32` documents the boot endpoints!).
Mid: DNS→TLS→HTML→JS→hydration→data fetching waterfall; that exemption comment is a rare map of an app's true boot sequence — cite it.
Senior: boot-loop failure modes (the comment literally explains an infinite-reload bug class) — turning trivia into an operations story.

## Q14: How would you type a function that takes a config and returns a decorated version of another function?
Testing: higher-order typing.
Anchor: the Python `validate_app_token` overloads (`service_api/wraps.py#L45-L57`) — decorator with/without args; TS analogue: `ParamSpec`-like `(...args: P) => R` preservation.
Mid: generic `<A extends unknown[], R>(fn: (...a: A) => R) => (...a: A) => R`.
Senior: when the wrapper *changes* the signature (kwargs injection!) the types must say so — and that's exactly the K5 kata critique.

---

**Cram usage**: shuffle, answer aloud, self-grade with the mid column as the pass bar and one senior element as stretch. Cards 1, 2, 4, 5, 8 are the most frequently asked — over-practice those.
