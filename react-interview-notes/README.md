# ⚛️ React Interview Prep — Complete Notes (4 YOE)

> Notes for every topic in `00-react-roadmap.md`, from basics to advanced, written for a **4-year experienced Frontend / React developer**.
>
> Each file follows the same structure: **concept → how it works internally → code → pitfalls → output/behaviour questions → interview answers → key takeaways → readiness checklist**.
>
> Written for **React 19.x** (with notes on what changed from 18), TanStack Query v5, Redux Toolkit 2, React Router v6.4+/v7 and the Next.js App Router.
>
> Prerequisite: the Advanced JavaScript notes (closures, `this`, promises, event loop, immutability).

---

## 📚 File Index

| # | File | Topic | Priority |
| --- | --- | --- | --- |
| 00 | [00-react-roadmap.md](00-react-roadmap.md) | Roadmap & progress tracker | — |
| 01 | [01-react-fundamentals.md](01-react-fundamentals.md) | What React is, Virtual DOM, render & commit, StrictMode | ⭐ |
| 02 | [02-jsx-conditional-rendering-lists.md](02-jsx-conditional-rendering-lists.md) | JSX, conditional rendering, lists & keys | 🔥 |
| 03 | [03-components-props-state.md](03-components-props-state.md) | Props, state as snapshot, lifting state, controlled/uncontrolled | 🔥 |
| 04 | [04-events-and-forms.md](04-events-and-forms.md) | Synthetic events, forms, React Hook Form + Zod, form actions | ⭐ |
| 05 | [05-core-hooks.md](05-core-hooks.md) | Rules of hooks, useState, useEffect, useRef, useLayoutEffect, useId | 🔥🔥 |
| 06 | [06-useeffect-deep-dive.md](06-useeffect-deep-dive.md) | Deps, cleanup, race conditions, "you might not need an effect" | 🔥🔥🔥 |
| 07 | [07-advanced-hooks.md](07-advanced-hooks.md) | useReducer, useMemo/useCallback, useSyncExternalStore, transitions, React 19 hooks | 🔥🔥 |
| 08 | [08-custom-hooks.md](08-custom-hooks.md) | 15 custom hooks implemented, latest-ref pattern | 🔥🔥 |
| 09 | [09-rendering-reconciliation-fiber.md](09-rendering-reconciliation-fiber.md) | Re-render triggers, reconciliation, Fiber, batching | 🔥🔥🔥 |
| 10 | [10-context-api.md](10-context-api.md) | Context, re-render pitfalls, optimization | 🔥 |
| 11 | [11-state-management.md](11-state-management.md) | State types, Zustand, Jotai, comparisons | 🔥 |
| 12 | [12-redux-toolkit.md](12-redux-toolkit.md) | Redux principles, RTK, selectors, thunks, RTK Query | 🔥 |
| 13 | [13-data-fetching-tanstack-query.md](13-data-fetching-tanstack-query.md) | TanStack Query: caching, mutations, optimistic, infinite | 🔥🔥🔥 |
| 14 | [14-react-router.md](14-react-router.md) | Nested routes, protected routes, loaders/actions, lazy | 🔥 |
| 15 | [15-performance-optimization.md](15-performance-optimization.md) | Profiling, memoization, React Compiler, splitting, virtualization | 🔥🔥🔥 |
| 16 | [16-component-patterns.md](16-component-patterns.md) | HOC, render props, compound, headless, controlled props, polymorphic | 🔥🔥 |
| 17 | [17-refs-portals-dom.md](17-refs-portals-dom.md) | Refs, ref as prop, callback refs, portals | ⭐ |
| 18 | [18-error-boundaries-suspense-lazy.md](18-error-boundaries-suspense-lazy.md) | Error boundaries, Suspense, lazy | 🔥 |
| 19 | [19-concurrent-react.md](19-concurrent-react.md) | Concurrent rendering, transitions, deferred values, Activity | 🔥 |
| 20 | [20-react-19-features.md](20-react-19-features.md) | Actions, new hooks, removed APIs, 19.2 features | 🔥🔥 |
| 21 | [21-ssr-hydration-server-components.md](21-ssr-hydration-server-components.md) | CSR/SSR/SSG/ISR, hydration, Server Components | 🔥🔥🔥 |
| 22 | [22-nextjs.md](22-nextjs.md) | Next.js App Router, data fetching, caching, Server Actions | 🔥🔥 |
| 23 | [23-typescript-with-react.md](23-typescript-with-react.md) | Typing props, events, hooks, reducers, generics | 🔥 |
| 24 | [24-styling.md](24-styling.md) | CSS Modules, Tailwind, CSS-in-JS, theming | ⭐ |
| 25 | [25-testing.md](25-testing.md) | RTL, Vitest, MSW, hooks, Playwright | 🔥 |
| 26 | [26-accessibility.md](26-accessibility.md) | Semantic HTML, ARIA, focus, keyboard, testing | 🔥 |
| 27 | [27-security.md](27-security.md) | XSS vectors, tokens, secrets, Server Action security | ⭐ |
| 28 | [28-architecture-best-practices.md](28-architecture-best-practices.md) | Folder structure, API layer, design systems, monorepos, micro-frontends | 🔥🔥 |
| 29 | [29-build-tools-tooling.md](29-build-tools-tooling.md) | Vite, webpack, SWC, ESLint, CI/CD | ⭐ |
| 30 | [30-class-components-legacy.md](30-class-components-legacy.md) | Class lifecycles, mapping to hooks | ⭐ |
| 31 | [31-interview-questions.md](31-interview-questions.md) | 100 rapid-fire Q&A + 10 scenario questions | 🔥🔥🔥 |
| 32 | [32-machine-coding.md](32-machine-coding.md) | 14 full solutions + 15 more specs | 🔥🔥🔥 |
| 33 | [33-frontend-system-design.md](33-frontend-system-design.md) | RADIO framework + 6 worked designs | 🔥🔥🔥 |

---

## 🎯 Study Plan (8 phases)

| Phase | Files | Goal |
| --- | --- | --- |
| 1 — Foundations | 01–04 | Mental model: components, props, state, events |
| 2 — Hooks | 05–08 | Master hooks, effects, custom hooks |
| 3 — Internals | 09, 10, 19 | Rendering, reconciliation, Fiber, context, concurrency |
| 4 — Ecosystem | 11–14 | State management, Redux, TanStack Query, Router |
| 5 — Performance & Patterns | 15–18 | Optimization, patterns, refs/portals, errors & Suspense |
| 6 — Modern React | 20–22 | React 19, SSR/RSC, Next.js |
| 7 — Professional | 23–30 | TypeScript, styling, testing, a11y, security, architecture, tooling, legacy |
| 8 — Interview Rounds | 31–33 | Q&A, machine coding, system design |

## ⏱️ Two-Week Crash Plan

| Day | Files |
| --- | --- |
| 1 | 01, 02, 03 |
| 2 | 05, 06 |
| 3 | 07, 08 |
| 4 | 09, 19 |
| 5 | 10, 11, 12 |
| 6 | 13, 14 |
| 7 | 15 + 32 (Todo, Autocomplete) |
| 8 | 16, 17, 18 |
| 9 | 20, 21 |
| 10 | 22, 23 |
| 11 | 25, 26, 27 |
| 12 | 28, 29, 30 |
| 13 | 32 (all remaining problems, timed) |
| 14 | 31 (all 100 out loud) + 33 (2 designs) |

---

## 🔁 Revision Method

- **Day 1:** read, type the examples yourself, build the checklist items
- **Day 3:** answer the "Interview Answers" out loud without looking
- **Day 7:** redo machine-coding problems from scratch with a timer
- **Day 30:** rapid-fire Q&A (file 31) + one system design

> At 4 YOE, interviewers expect **why** (internals), **trade-offs**, and **real project stories**. For every topic, prepare one "In my last project…" example.
