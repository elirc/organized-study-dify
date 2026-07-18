# Frontend Framework Cards (React 19 / Next / State / Perf)

## Q1: When does a React component re-render, and how do you stop unnecessary ones?
Anchor: chat streaming re-renders (`chat/hooks.ts` callbacks mutating tree state); workflow canvas Zustand selectors.
Junior: state/props change.
Mid: parent re-render cascades; `memo` + stable props; Zustand *selector subscriptions* so a node component ignores unrelated store changes; concrete: dragging one workflow node shouldn't re-render 200 others.
Senior: measure first (Profiler), then batch at the source — coalesce SSE chunks per frame rather than memoizing 50 components.
Follow-up: why is `memo` useless if you pass a fresh object literal?

## Q2: Where should state live? Walk your decision tree.
Anchor: three-way split — chat tree in hook state, canvas in Zustand slices (`workflow/store/workflow/`), server data in TanStack Query (`use-apps.ts`).
Mid: ephemeral+single-owner → useState; shared tool-state → store; anything fetched → query cache; URL-worthy → params (`nuqs` is in the catalog).
Senior: the *migration costs* between the categories, and misclassification smells (server data copied into useState = staleness bugs).

## Q3: useEffect: what is it actually for?
Anchor: the codebase's data layer means effects rarely fetch; effects synchronize with external systems (subscriptions, DOM, SSE).
Mid: "synchronize with something outside React"; cleanup functions; dependency honesty; fetch-in-effect replaced by Query.
Senior: effect-free alternatives (event handlers own writes; derived state during render) and React 19 context.

## Q4: How do keys work and when do they bite?
Anchor: chat tree items keyed by message id — with `Date.now()` placeholder ids replaced by server ids (Pattern 14, `chat/hooks.ts#L644-L679`).
Mid: identity across renders; index-keys break on reorder/insert; the placeholder→real-id swap is exactly where a wrong key would remount and flicker.
Senior: keys as *state-preservation* boundaries, deliberate remount via key change.

## Q5: Explain TanStack Query's caching model.
Anchor: `use-apps.ts#L30-L81`.
Junior: it caches requests.
Mid: keys as cache addresses; stale-while-revalidate; invalidation on mutation (`#L70-L78`); `enabled` gating; retries/dedup free.
Senior: key-factory governance at scale — the dual-key migration risk in this repo and how centralized factories prevent it.

## Q6: Optimistic updates — how, and what about failure?
Anchor: chat placeholders (`#L644-L668`), M5 ticket design.
Mid: insert temp entity → reconcile on success → roll back or mark errored on failure; show the placeholder-orphan risk.
Senior: idempotency keys for retried writes; when optimistic is wrong (money, destructive ops).

## Q7: How do you handle auth in a React SPA?
Anchor: cookie session + CSRF header echo (`fetch.ts#L184`), 401 → refresh single-flight → retry (`base.ts#L540-L545`), passport JWT for public share pages (`#L502-L506`).
Mid: httpOnly cookies over localStorage tokens (XSS); CSRF needed with cookies; refresh rotation.
Senior: multi-surface auth (console vs public webapp) with different threat models in one app.

## Q8: Your app is slow. Walk me through frontend performance work.
Anchor: [05/04-performance-thinking](../05-quality-engineering/04-performance-thinking.md) hotspots — streaming re-renders, canvas selectors, heavyweight libs (echarts/mermaid/monaco) needing dynamic import.
Mid: measure (Profiler, network, bundle analyzer) → classify (render/network/bundle) → cheapest fix; one concrete story per class.
Senior: perf budgets and regression guards, not one-off heroics.

## Q9: Explain SSR/RSC boundaries in a Next app.
Anchor: `web/app/` app router; heavy interactivity (canvas, chat) is client components; `web/env.ts` split env.
Mid: server components render ahead of hydration, can't hold browser state; client islands; secret leakage rules.
Senior: this repo runs Next through vinext/vite-plus — being able to say "I check the build tooling before assuming framework defaults" is itself the answer.

## Q10: How do you structure the API layer of a large React app?
Anchor: `web/service/` — one configured client (`fetch.ts#L122-L205`), resource modules, query-hook modules, typed responses, global error policy.
Mid: centralize cross-cutting (auth, CSRF, errors); components never import fetch.
Senior: policy placement critique — global toasts in the HTTP layer (`#L61-L79`) and its `silent` escape hatch; codegen direction.

## Q11: Forms: controlled vs uncontrolled, validation strategy.
Anchor: `@tanstack/react-form` in catalog; app config forms under `web/app/components/app/configuration/`.
Mid: controlled for dependent/validated fields; schema validation (zod 4 in catalog) shared with API expectations.
Senior: form state as a *contract preview* — same schema on client and server, single source.

## Q12: Accessibility — what do you actually do?
Anchor: overlay primitives centralized in `@langgenius/dify-ui` (`web/AGENTS.md` mandates them) — focus traps/portals/z-layers handled once.
Mid: semantic elements, keyboard paths, focus management in modals; centralizing overlays so a11y is fixed in one place.
Senior: a11y as a design-system contract with lint enforcement, not per-feature heroics.

---

**Cram usage**: Q1, Q2, Q5, Q6, Q7 are near-guaranteed in mid-level fullstack loops. For each, prepare the 90-second answer with the Dify anchor named aloud.
