# 31 — Rapid-Fire React Interview Q&A 🔥🔥🔥

> **Interview + Understanding Guide** — Roadmap Section 31
>
> 100 questions with concise, interview-ready answers. Cover the answer, say it out loud, then check. Each section links to the deep-dive file.

---

## A. Fundamentals (01–03)

**1. What is React?**
A declarative, component-based JavaScript library for building UIs; you describe UI for a given state and React updates the DOM efficiently.

**2. What is the Virtual DOM?**
The tree of React elements components return. React diffs new vs previous trees and commits minimal DOM changes. Its value is the declarative model, not raw speed.

**3. Element vs component?**
A component is a function returning UI; an element is the plain object `{ type, props, key }` describing what to render.

**4. What does JSX compile to?**
`jsx()` calls from `react/jsx-runtime` (or `React.createElement`) that create elements.

**5. Why must component names be capitalized?**
Lowercase tags are treated as DOM elements; capitalized identifiers are component references.

**6. Props vs state?**
Props: read-only inputs from the parent. State: data owned by the component that changes over time and triggers re-renders.

**7. Why is state a snapshot?**
Each render has constant state values; setters schedule a new render rather than mutating the current variable.

**8. Why use functional updates?**
When the next state depends on the previous state — multiple updates in one handler, async callbacks, intervals.

**9. Why must you not mutate state?**
React compares by reference; mutation keeps the same reference so React may skip re-rendering, and it breaks memoization and time-travel debugging.

**10. What is lifting state up?**
Moving shared state to the closest common parent and passing it down with callbacks — single source of truth.

**11. What is derived state and why avoid storing it?**
Values computable from props/state. Storing them duplicates data and causes sync bugs; compute during render (or `useMemo`).

**12. Controlled vs uncontrolled components?**
Controlled: value from React state + `onChange`. Uncontrolled: DOM keeps the value; read via ref or `FormData`.

**13. Why are keys needed in lists?**
To give items stable identity so React matches them across renders — correct state, minimal DOM operations.

**14. Why not use index as key?**
Inserting/reordering shifts indexes, attaching state/DOM to the wrong items. OK only for static lists.

**15. How do you reset a component's state?**
Change its `key`, which remounts it.

**16. What renders for `{0 && <X />}`?**
`0` — numbers render. Use `count > 0 &&`.

**17. What is a Fragment?**
`<>…</>` groups children without adding a DOM node; use `<Fragment key>` in lists.

**18. What is StrictMode?**
Dev-only checks: double render, double effect setup/cleanup, deprecated API warnings — to surface impure code and missing cleanups.

---

## B. Hooks (05–08)

**19. Rules of hooks?**
Call at the top level only, and only from components or custom hooks.

**20. Why do these rules exist?**
React stores hooks by call order in a list on the fiber; changing order corrupts state.

**21. Lazy initial state?**
`useState(() => expensive())` — the function runs only on the first render.

**22. useEffect dependency array options?**
None: every render. `[]`: mount only. `[a, b]`: when a or b change (`Object.is`).

**23. When does useEffect run?**
After commit, typically after paint. Cleanup runs before the next effect and on unmount.

**24. useEffect vs useLayoutEffect?**
Layout effect runs synchronously after DOM mutation, before paint — for measurements to avoid flicker; it blocks paint.

**25. Why does my effect run twice in development?**
StrictMode mount → unmount → mount to verify cleanup. Not in production.

**26. How do you fetch data in useEffect correctly?**
Inner async function, handle loading/error, ignore flag or AbortController in cleanup; better: TanStack Query.

**27. What causes infinite loops in effects?**
Setting state without deps, or depending on objects/functions recreated every render, or setting a state that's also a dependency.

**28. Name cases where you don't need an effect.**
Derived data, expensive calculations (useMemo), resetting state on prop change (key), user-event logic (handlers), notifying parents (call in handler), subscribing to stores (useSyncExternalStore).

**29. useRef use cases?**
DOM access; mutable values that persist without re-rendering (timer IDs, previous values, latest callback).

**30. useRef vs useState?**
Both persist; only state changes trigger re-renders.

**31. useMemo vs useCallback?**
useMemo caches a value; useCallback caches a function reference.

**32. When are useMemo/useCallback useless?**
Cheap computations; values passed to non-memoized children or DOM elements; deps that change every render.

**33. useReducer vs useState?**
Reducer for complex related state and many update types; centralizes logic, testable, stable dispatch.

**34. What is useContext's re-render behaviour?**
All consumers re-render when the provider value changes; memo doesn't prevent it.

**35. What is useImperativeHandle?**
Customizes the value exposed via ref — a limited imperative API.

**36. What is useSyncExternalStore?**
Tear-free subscription to external stores under concurrent rendering; used by Redux/Zustand.

**37. useTransition vs useDeferredValue?**
Transition wraps a state update you control (gives isPending); deferred value lags a received value (memoize the child).

**38. What does useId do?**
Stable, SSR-safe unique IDs for accessibility attributes.

**39. What is a custom hook?**
A `use`-prefixed function composing hooks to reuse stateful logic; each caller gets independent state.

**40. What is a stale closure?**
A long-lived callback capturing an old render's values. Fix with functional updates, correct deps, or the latest-ref pattern / `useEffectEvent`.

**41. Implement useDebounce in one sentence.**
Store a debounced copy in state, set it in a timeout inside an effect keyed on value/delay, clear the timeout in cleanup.

**42. What is `use` in React 19?**
Reads a promise (suspends) or context during render; can be called conditionally.

**43. useActionState / useFormStatus / useOptimistic?**
State + pending for actions; pending status of the parent form; temporary optimistic state during an action.

**44. What is useEffectEvent?**
Creates a function that reads latest props/state inside an effect without being a dependency (stable in 19.2).

---

## C. Rendering & Internals (09, 19)

**45. When does a component re-render?**
Own state change, parent re-render, consumed context change.

**46. Does a prop change cause a re-render?**
Props change because the parent re-rendered; the parent's re-render is the cause. Children re-render even with the same props unless memoized.

**47. Render phase vs commit phase?**
Render: call components & diff (pure, interruptible). Commit: apply DOM changes, refs, layout effects, then effects.

**48. What is reconciliation?**
Diffing new vs old element trees with O(n) heuristics: different type → remount; same type → update; keys for lists.

**49. What happens if you change an element's type at the same position?**
The subtree unmounts (state lost) and a new one mounts.

**50. What is Fiber?**
React's reconciler since v16: units of work in a linked tree enabling pausing, prioritizing (lanes), and double-buffered trees.

**51. What is automatic batching?**
React 18+ batches state updates from any source (events, promises, timeouts) into one render. `flushSync` opts out.

**52. What is concurrent rendering?**
Interruptible, prioritized rendering on one thread; opt-in via transitions; powers Suspense streaming and selective hydration.

**53. What is tearing?**
Different components showing different versions of external store data during a concurrent render; prevented by useSyncExternalStore.

**54. Why must render be pure?**
React may call it multiple times, pause, or discard it; side effects would run unpredictably.

**55. Does React re-render the DOM on every render?**
No. It re-runs components, but only commits differences to the DOM.

---

## D. Context & State Management (10–13)

**56. When to use Context?**
Rarely changing global-ish data: theme, auth, locale, DI. Not for high-frequency updates or server cache.

**57. How to optimize Context?**
Memoize the value, split contexts (state vs dispatch, by domain), keep fast-changing state elsewhere, use stores with selectors.

**58. Types of state?**
Local UI, shared client, server, URL, form, derived.

**59. Why separate server state?**
It's a cache of remote data needing refetch, dedupe, invalidation, background updates — handled by TanStack Query/RTK Query.

**60. Redux core principles?**
Single store, read-only state changed via actions, pure reducers.

**61. Why Redux Toolkit?**
Less boilerplate: createSlice + Immer, configureStore with devtools/thunk, createAsyncThunk, RTK Query.

**62. How to avoid re-renders with useSelector?**
Select minimal data; avoid returning new objects; memoize with createSelector; shallowEqual.

**63. Zustand vs Redux?**
Zustand: tiny, no provider, minimal boilerplate. Redux: structure, middleware, devtools, RTK Query; better for large teams.

**64. staleTime vs gcTime?**
staleTime: how long data is fresh (no refetch). gcTime: how long unused data stays in cache.

**65. How do optimistic updates work in TanStack Query?**
onMutate: cancel queries, snapshot, set optimistic data, return context; onError: rollback; onSettled: invalidate.

**66. What is a query key?**
Array uniquely identifying cached data; must include all inputs to the query function.

---

## E. Routing, Performance & Patterns (14–18)

**67. How does client-side routing work?**
Intercept links, update URL via History API, render matching routes, listen to popstate.

**68. How to implement protected routes?**
Guard/layout route checking auth → Outlet or Navigate (preserving location); or loader redirect. Server enforces authorization.

**69. What are loaders/actions in React Router?**
Route-level data fetching before render and form mutation handlers with automatic revalidation.

**70. How do you find performance problems?**
React Profiler ("why did this render"), Performance panel on prod build with CPU throttling, Web Vitals.

**71. First things to try before memoization?**
Move state down, pass expensive subtrees as children, split context, remove unnecessary effects.

**72. Why might React.memo not work?**
Props are new references each render (inline functions/objects/children) or context changes.

**73. What is the React Compiler?**
Build-time plugin auto-memoizing components/values for code following the Rules of React.

**74. What is code splitting?**
Splitting the bundle into chunks loaded on demand via dynamic `import()` / `React.lazy` — route-based first.

**75. What is virtualization?**
Rendering only visible list items (TanStack Virtual, react-window) for large lists.

**76. HOC vs render props vs hooks?**
HOC wraps and injects props (wrapper hell, collisions); render props pass state to a render function; hooks share logic flatly — default choice.

**77. What are compound components?**
Related components sharing implicit state via context (`Tabs`, `Tabs.Tab`), giving flexible markup.

**78. What is a headless component?**
Provides behaviour/state/a11y without styles; consumer renders markup (Radix, React Aria, Downshift).

**79. What are portals and how do events behave?**
Render into another DOM node; still in React tree — context works and events bubble through React ancestors.

**80. What is forwardRef / ref as prop?**
Passing refs to child DOM nodes; React 19 accepts `ref` as a normal prop for function components.

**81. What do error boundaries not catch?**
Event handlers, async code outside render, SSR errors, errors in the boundary itself.

**82. How does Suspense work?**
A component suspends (lazy load, `use(promise)`, Suspense-enabled library); nearest boundary shows fallback until resolved.

---

## F. SSR, Next.js, React 19 (20–22)

**83. CSR vs SSR vs SSG vs ISR?**
Browser render; per-request server render; build-time render; static with background regeneration.

**84. What is hydration?**
Attaching React event handlers/state to server-rendered HTML by rendering the same tree on the client.

**85. Causes of hydration mismatch?**
Browser-only APIs, dates/random values, locale/timezone, invalid HTML nesting, extensions.

**86. Server vs Client Components?**
Server: run only on server, async, direct data access, zero client JS, no state/effects. Client (`"use client"`): hydrated, interactive.

**87. RSC vs SSR?**
SSR renders all components to HTML and hydrates them all; RSC components never ship JS. They're complementary.

**88. Can a Client Component import a Server Component?**
No — but it can receive one as `children`/props from a Server Component.

**89. What are Server Actions? Security concerns?**
Server functions callable from the client/forms; they're public endpoints — validate, authenticate, authorize.

**90. How does Next.js caching/revalidation work?**
Request memoization, data cache, full-route cache, router cache; time-based (`revalidate`) or on-demand (`revalidatePath/Tag`); defaults changed in v15/16.

**91. Main React 19 features?**
Actions, useActionState/useFormStatus/useOptimistic, use, Server Components/Actions, ref as prop, Context as provider, metadata in components, removed legacy APIs.

---

## G. TypeScript, Testing, A11y, Security, Architecture (23–29)

**92. How to type children?**
`React.ReactNode`.

**93. How to extend native element props?**
`React.ComponentPropsWithoutRef<"button"> & MyProps`.

**94. Testing philosophy with RTL?**
Test behaviour like a user: query by role/label, interact with userEvent, assert visible output; mock network with MSW.

**95. getBy vs queryBy vs findBy?**
Throws if missing / returns null / async retry.

**96. How do you make a modal accessible?**
role dialog, aria-modal, labelled title, focus in, trap Tab, Esc closes, inert background, return focus.

**97. Is React XSS-safe?**
Escapes JSX text, but not `dangerouslySetInnerHTML`, `javascript:` URLs, refs/innerHTML, eval, untrusted prop spreading.

**98. Where to store auth tokens?**
HttpOnly Secure SameSite cookie (with CSRF protection); avoid localStorage for sensitive tokens.

**99. How do you structure a large React app?**
Feature-based folders with public APIs, shared UI/lib layer, typed API client, server state in Query hooks, enforced boundaries.

**100. Vite vs webpack?**
Vite: native-ESM dev server (fast startup/HMR), Rollup prod build. webpack: bundles in dev, more configurable, Module Federation.

---

## H. Scenario Questions (practise talking for 2–3 minutes each)

1. "Our dashboard is slow when typing in the filter box." → Profile → colocate → memo/useMemo → useDeferredValue → virtualize → debounce server calls.
2. "Users see the wrong user's profile briefly after switching quickly." → race condition → abort/ignore flag → TanStack Query.
3. "How would you migrate a large CRA app?" → Vite migration, env var prefixes, tests, incremental.
4. "Design a reusable Select component for the design system." → headless hook, controlled/uncontrolled, a11y (listbox/combobox), keyboard, portal, virtualization for large options, TS generics.
5. "How would you add SSR to an existing SPA?" → evaluate needs (SEO, LCP), Next.js migration path route by route, hydration issues, data fetching changes.
6. "A memory leak report: the app slows after navigating many times." → heap snapshots, missing effect cleanups, listeners, intervals, subscriptions, caches.
7. "How do you handle global error reporting?" → error boundaries per level, Sentry, onUncaughtError/onCaughtError, API error mapping, toasts.
8. "Tell me about a hard bug you fixed." → STAR format: situation, investigation (tools), root cause, fix, prevention (tests/lint).
9. "How do you decide between Context, Zustand and Redux?" → state classification + team/scale trade-offs.
10. "How do you ensure quality in a team?" → TS strict, lint, tests (trophy), PR reviews, CI gates, Storybook, ADRs.
