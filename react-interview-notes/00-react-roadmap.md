# ⚛️ React Interview Roadmap — 4 YOE Frontend Developer

> A structured React preparation track for a **4-year experienced Frontend / React Developer**.
>
> **Goal:** Master React from fundamentals to internals, the ecosystem (routing, state, data fetching, Next.js, TypeScript, testing), performance, architecture, machine coding and frontend system design.
>
> **Prerequisite:** the Advanced JavaScript notes (closures, `this`, promises, event loop, immutability).

---

## 📊 Progress Tracker

| # | Section | File | Status |
| --- | --- | --- | --- |
| 1 | React Fundamentals | `01-react-fundamentals.md` | ⬜ Not Started |
| 2 | JSX & Rendering Lists | `02-jsx-conditional-rendering-lists.md` | ⬜ Not Started |
| 3 | Components, Props & State | `03-components-props-state.md` | ⬜ Not Started |
| 4 | Events & Forms | `04-events-and-forms.md` | ⬜ Not Started |
| 5 | Core Hooks | `05-core-hooks.md` | ⬜ Not Started |
| 6 | useEffect Deep Dive | `06-useeffect-deep-dive.md` | ⬜ Not Started |
| 7 | Advanced Hooks | `07-advanced-hooks.md` | ⬜ Not Started |
| 8 | Custom Hooks | `08-custom-hooks.md` | ⬜ Not Started |
| 9 | Rendering, Reconciliation & Fiber | `09-rendering-reconciliation-fiber.md` | ⬜ Not Started |
| 10 | Context API | `10-context-api.md` | ⬜ Not Started |
| 11 | State Management | `11-state-management.md` | ⬜ Not Started |
| 12 | Redux Toolkit & RTK Query | `12-redux-toolkit.md` | ⬜ Not Started |
| 13 | Data Fetching & TanStack Query | `13-data-fetching-tanstack-query.md` | ⬜ Not Started |
| 14 | React Router | `14-react-router.md` | ⬜ Not Started |
| 15 | Performance Optimization | `15-performance-optimization.md` | ⬜ Not Started |
| 16 | Component Design Patterns | `16-component-patterns.md` | ⬜ Not Started |
| 17 | Refs, Portals & DOM | `17-refs-portals-dom.md` | ⬜ Not Started |
| 18 | Error Boundaries, Suspense & Lazy | `18-error-boundaries-suspense-lazy.md` | ⬜ Not Started |
| 19 | Concurrent React | `19-concurrent-react.md` | ⬜ Not Started |
| 20 | React 19 Features | `20-react-19-features.md` | ⬜ Not Started |
| 21 | SSR, Hydration & Server Components | `21-ssr-hydration-server-components.md` | ⬜ Not Started |
| 22 | Next.js (App Router) | `22-nextjs.md` | ⬜ Not Started |
| 23 | TypeScript with React | `23-typescript-with-react.md` | ⬜ Not Started |
| 24 | Styling in React | `24-styling.md` | ⬜ Not Started |
| 25 | Testing React | `25-testing.md` | ⬜ Not Started |
| 26 | Accessibility | `26-accessibility.md` | ⬜ Not Started |
| 27 | Security in React | `27-security.md` | ⬜ Not Started |
| 28 | Architecture & Best Practices | `28-architecture-best-practices.md` | ⬜ Not Started |
| 29 | Build Tools & Tooling | `29-build-tools-tooling.md` | ⬜ Not Started |
| 30 | Class Components & Legacy React | `30-class-components-legacy.md` | ⬜ Not Started |
| 31 | Rapid-Fire Interview Q&A | `31-interview-questions.md` | ⬜ Not Started |
| 32 | Machine Coding Round | `32-machine-coding.md` | ⬜ Not Started |
| 33 | Frontend System Design | `33-frontend-system-design.md` | ⬜ Not Started |

### Legend

- ⬜ Not Started · 🔄 In Progress · ✅ Completed · ⭐ High Priority · 🔥 Very High Priority

---

# 1. React Fundamentals

- [ ] What React is & why (declarative, component-based, unidirectional data flow)
- [ ] Library vs framework
- [ ] ⭐ Virtual DOM — what it is, what it is NOT
- [ ] `createRoot`, `root.render`, `StrictMode`
- [ ] React element vs component vs instance
- [ ] ⭐ Render = calling your component; commit = updating the DOM

# 2. JSX, Conditional Rendering & Lists

- [ ] JSX → `jsx()` / `React.createElement` (new JSX transform)
- [ ] Expressions, attributes (`className`, `htmlFor`, `style`)
- [ ] Fragments
- [ ] Conditional rendering (`&&` 0-trap, ternary, early return)
- [ ] ⭐⭐⭐ Lists & keys (why not index)

# 3. Components, Props & State

- [ ] Function components
- [ ] ⭐ Props (read-only), `children`, default values
- [ ] ⭐⭐⭐ State as a snapshot
- [ ] Lifting state up · single source of truth
- [ ] ⭐ Derived state (don't store what you can compute)
- [ ] Controlled vs uncontrolled components
- [ ] Composition vs inheritance
- [ ] Resetting state with `key`

# 4. Events & Forms

- [ ] Synthetic events, event delegation at root
- [ ] Passing arguments to handlers
- [ ] ⭐ Controlled forms · uncontrolled forms with `FormData`
- [ ] Validation · React Hook Form + Zod
- [ ] Form actions (React 19)

# 5. Core Hooks 🔥

- [ ] ⭐⭐⭐ Rules of hooks & why
- [ ] ⭐⭐⭐ `useState` (lazy init, functional updates, batching)
- [ ] ⭐⭐⭐ `useEffect`
- [ ] ⭐⭐⭐ `useRef`
- [ ] `useLayoutEffect` vs `useEffect`
- [ ] `useId`

# 6. useEffect Deep Dive 🔥🔥🔥

- [ ] ⭐⭐⭐ Dependency array semantics
- [ ] ⭐⭐⭐ Cleanup
- [ ] ⭐⭐⭐ Race conditions
- [ ] ⭐⭐⭐ "You might not need an effect"
- [ ] StrictMode double invoke
- [ ] Infinite loop causes

# 7. Advanced Hooks 🔥

- [ ] ⭐⭐⭐ `useReducer`
- [ ] ⭐⭐⭐ `useContext`
- [ ] ⭐⭐⭐ `useMemo` / `useCallback`
- [ ] `useImperativeHandle`
- [ ] `useSyncExternalStore`
- [ ] ⭐ `useTransition` / `useDeferredValue`
- [ ] `useInsertionEffect`, `useDebugValue`
- [ ] React 19: `use`, `useActionState`, `useOptimistic`, `useFormStatus`, `useEffectEvent`

# 8. Custom Hooks 🔥

- [ ] ⭐⭐⭐ Rules & naming
- [ ] useFetch · useDebounce · useThrottle · usePrevious · useLocalStorage
- [ ] useToggle · useInterval · useEventListener · useOnClickOutside
- [ ] useMediaQuery · useIntersectionObserver · useWindowSize

# 9. Rendering, Reconciliation & Fiber 🔥🔥🔥

- [ ] ⭐⭐⭐ When does a component re-render?
- [ ] ⭐⭐⭐ Render phase vs commit phase
- [ ] ⭐⭐⭐ Reconciliation & diffing heuristics
- [ ] ⭐ Fiber architecture
- [ ] ⭐ Automatic batching
- [ ] Bail-outs

# 10. Context API

- [ ] ⭐ createContext / Provider / useContext
- [ ] ⭐⭐⭐ Context performance pitfalls & fixes
- [ ] Context vs prop drilling vs composition vs state libraries

# 11–14. Ecosystem 🔥

- [ ] ⭐ State categories (local, global, server, URL, form)
- [ ] ⭐ Redux Toolkit (slices, thunks, selectors, RTK Query)
- [ ] Zustand · Jotai · comparison
- [ ] ⭐⭐⭐ TanStack Query (cache, staleTime, invalidation, mutations, optimistic, infinite)
- [ ] ⭐ React Router (nested routes, params, loaders/actions, protected routes, lazy)

# 15. Performance 🔥🔥🔥

- [ ] ⭐⭐⭐ React.memo / useMemo / useCallback — when they help
- [ ] ⭐⭐⭐ State colocation, lifting content up
- [ ] ⭐ React Profiler · why-did-you-render
- [ ] ⭐ Code splitting · lazy · virtualization
- [ ] React Compiler

# 16–20. Advanced React 🔥

- [ ] ⭐ Patterns: composition, HOC, render props, compound components, headless, controlled props, polymorphic
- [ ] ⭐ Refs, forwardRef → ref as prop, callback refs, portals
- [ ] ⭐ Error boundaries · Suspense · lazy
- [ ] ⭐ Concurrent rendering · transitions
- [ ] ⭐ React 19: Actions, `use`, Server Components, Compiler

# 21–22. SSR & Next.js 🔥

- [ ] ⭐⭐⭐ CSR vs SSR vs SSG vs ISR vs RSC
- [ ] ⭐⭐⭐ Hydration & hydration errors
- [ ] ⭐ Server Components vs Client Components
- [ ] ⭐ Next.js App Router, layouts, data fetching, caching, Server Actions, middleware

# 23–30. Professional Skills

- [ ] ⭐ TypeScript with React
- [ ] Styling approaches & trade-offs
- [ ] ⭐ Testing: RTL, Vitest/Jest, MSW, Playwright
- [ ] ⭐ Accessibility
- [ ] Security
- [ ] ⭐ Architecture: folder structure, design systems, micro-frontends
- [ ] Tooling: Vite, webpack, SWC, ESLint, CI
- [ ] Class components & lifecycle mapping (legacy interviews)

# 31–33. Interview Rounds 🔥🔥🔥

- [ ] ⭐⭐⭐ 100 rapid-fire questions
- [ ] ⭐⭐⭐ Machine coding: 15+ components
- [ ] ⭐⭐⭐ Frontend system design (RADIO framework)

---

# 🎯 Recommended Learning Order

| Phase | Files | Goal |
| --- | --- | --- |
| 1 — Foundations | 01–04 | Mental model: components, props, state, events |
| 2 — Hooks | 05–08 | Master hooks, effects, custom hooks |
| 3 — Internals | 09, 10, 19 | Rendering, reconciliation, Fiber, context, concurrency |
| 4 — Ecosystem | 11–14 | State, Redux, TanStack Query, Router |
| 5 — Performance & Patterns | 15–18 | Optimisation, patterns, refs, error handling |
| 6 — Modern React | 20–22 | React 19, SSR, RSC, Next.js |
| 7 — Professional | 23–30 | TypeScript, styling, testing, a11y, security, architecture |
| 8 — Interview Rounds | 31–33 | Q&A, machine coding, system design |

---

# 🏆 4-YOE Must-Master List

1. [ ] Rendering behaviour: when & why components re-render
2. [ ] Reconciliation, keys, Fiber
3. [ ] State as snapshot, batching, functional updates
4. [ ] useEffect: deps, cleanup, race conditions, avoiding unnecessary effects
5. [ ] Stale closures
6. [ ] useMemo / useCallback / React.memo trade-offs
7. [ ] useReducer + Context architecture
8. [ ] Context performance
9. [ ] Custom hooks design
10. [ ] Server state vs client state (TanStack Query)
11. [ ] Redux Toolkit
12. [ ] Code splitting & lazy loading
13. [ ] Virtualization
14. [ ] Error boundaries & Suspense
15. [ ] Concurrent features (useTransition, useDeferredValue)
16. [ ] SSR, hydration, Server Components
17. [ ] Next.js App Router & caching
18. [ ] Component patterns (compound, headless, render props, HOC)
19. [ ] Controlled vs uncontrolled
20. [ ] Forms at scale (React Hook Form + Zod)
21. [ ] TypeScript generics in components
22. [ ] Testing with RTL + MSW
23. [ ] Accessibility (focus, ARIA, keyboard)
24. [ ] XSS & auth in React apps
25. [ ] Folder structure & scalable architecture
26. [ ] Micro-frontends basics
27. [ ] Performance debugging with Profiler & Web Vitals
28. [ ] React 19 Actions & Compiler
29. [ ] Machine coding under time pressure
30. [ ] Frontend system design

---

# 🧪 Topic Readiness Checklist

A topic is ✅ only when you can:

- [ ] Explain it in 30 seconds and in 2 minutes
- [ ] Explain **why** React works that way (internals)
- [ ] Write it from scratch without docs
- [ ] Debug a broken example
- [ ] Discuss trade-offs and alternatives
- [ ] Give a real project example ("In my last project we…")

> At 4 YOE, interviewers expect **trade-offs, debugging stories and architecture decisions**, not just definitions.
