# 24 — Angular vs React 🔥

> **Interview + Understanding Guide** — Roadmap Section 24
>
> Goal: Compare Angular and React fairly and concretely — architecture, reactivity, rendering, state, forms, tooling — and map concepts between them (handy if you know one and interview for the other).

---

# 1. Big Picture ⭐

| | Angular | React |
| --- | --- | --- |
| Type | **Full framework** (batteries included) | **UI library** (+ ecosystem / meta-frameworks like Next.js) |
| Maintainer | Google | Meta |
| Language | TypeScript-first | JavaScript/TypeScript, JSX |
| Templates | HTML templates + Angular syntax (`@if`, bindings) | JSX (JavaScript expressions) |
| Reactivity | **Signals** (fine-grained) + RxJS | State + re-render component, **virtual DOM** diffing (React Compiler auto-memoizes) |
| DI | Built-in hierarchical DI | None built in (Context, props, module imports) |
| Routing / Forms / HTTP | Built in (`@angular/router`, `forms`, `common/http`) | Third-party (React Router/TanStack Router, React Hook Form, fetch/TanStack Query) |
| SSR | `@angular/ssr`, hybrid render modes, incremental hydration | Next.js / Remix (React Router), RSC & Server Actions |
| Structure | Opinionated, consistent across teams | Flexible, team decides |
| CLI | Angular CLI (generate, build, test, update migrations) | Vite / framework CLIs |
| Releases | Yearly majors (from v22) with automated migrations | Less frequent majors |
| Learning curve | Steeper upfront (more concepts), simpler since signals/standalone | Easier start, complexity grows with ecosystem choices |

---

# 2. Concept Mapping ⭐⭐⭐

| Concept | Angular | React |
| --- | --- | --- |
| Component | `@Component` class + template | Function component returning JSX |
| Local state | `signal(0)` | `useState(0)` |
| Derived state | `computed(() => …)` | Compute during render / `useMemo` |
| Side effects | `effect()`, `afterNextRender` | `useEffect`, `useLayoutEffect` |
| Writable derived | `linkedSignal` | `useState` + reset with `key` / derive during render |
| Props in | `input()` | props |
| Events out | `output()` | callback props (`onChange`) |
| Two-way | `model()` + `[(x)]` | value + onChange (controlled) |
| Children / slots | `<ng-content>` (multi-slot `select`) | `children`, render props |
| Refs | `viewChild()`, template refs `#ref` | `useRef` |
| Context / shared | DI services (`inject()`), component providers | Context API, Zustand/Redux |
| Conditional | `@if` | `{cond && …}`, ternary |
| Lists | `@for (… ; track id)` | `.map()` with `key` |
| Lazy component | `@defer`, `loadComponent` | `React.lazy` + `Suspense` |
| Data fetching | `httpResource`, HttpClient + RxJS | TanStack Query, `use()`, RSC |
| Memoization | Pure pipes, `computed`, OnPush | `memo`, `useMemo`, `useCallback`, React Compiler |
| Error boundaries | `ErrorHandler`, `@defer` `@error` (`@boundary` coming) | Error Boundaries |
| Forms | Reactive / Signal Forms / template-driven | Controlled inputs, React Hook Form, Actions + `useActionState` |
| Global state | Signal services, NgRx, SignalStore | Redux Toolkit, Zustand, Jotai |
| Lifecycle | `ngOnInit`, `ngOnDestroy`, `DestroyRef` | `useEffect` setup/cleanup |
| Styling scope | View encapsulation (emulated) | CSS Modules, CSS-in-JS, Tailwind |
| Testing | TestBed + Vitest, harnesses | React Testing Library + Vitest/Jest |

### Same feature in both

```ts
// Angular
@Component({
  selector: "app-counter",
  template: `<button (click)="count.update(c => c + 1)">Clicked {{ count() }} times ({{ double() }})</button>`,
})
export class Counter {
  protected readonly count = signal(0);
  protected readonly double = computed(() => this.count() * 2);
}
```

```tsx
// React
function Counter() {
  const [count, setCount] = useState(0);
  const double = count * 2;
  return <button onClick={() => setCount((c) => c + 1)}>Clicked {count} times ({double})</button>;
}
```

---

# 3. Rendering & Reactivity Differences ⭐⭐

**React:** state change → component function **re-runs** → new virtual DOM → diff → patch DOM. Children re-render by default unless memoized (React Compiler automates much of this). Rules of Hooks; stale closures are a classic pitfall.

**Angular:** templates compiled to instructions; with signals, a change marks only the **consuming view** dirty and updates the specific bindings — no virtual DOM, components aren't re-executed (class instance persists). Pitfalls: forgetting `()` on signals, mutating state, effects for derived state, legacy zone/OnPush confusion.

| | Angular | React |
| --- | --- | --- |
| Unit of re-execution | Template bindings of dirty views | Whole component function |
| DOM update | Direct, compiled instructions | Virtual DOM diff |
| Dependency tracking | Automatic (signals) | Manual dependency arrays (`useEffect`, `useMemo`), compiler helps |
| Component instance | Persistent class | Function re-called; state in hooks |

---

# 4. When to Choose Which ⭐

**Angular fits when:**
- Large enterprise apps, many teams — consistency and conventions matter
- You want routing, forms, HTTP, DI, testing, i18n, SSR **officially integrated** and upgraded together (`ng update` migrations)
- Strong TypeScript and OOP/DI background; complex forms (admin panels, banking, ERP)

**React fits when:**
- You want flexibility and the largest ecosystem/hiring pool
- Content sites / full-stack apps with Next.js (RSC, Server Actions)
- Teams comfortable picking and maintaining their own stack; React Native for mobile reuse

Both are excellent, performant and modern; the choice is mostly about team, constraints and ecosystem.

---

# 5. Interview Answers

**Angular vs React?**

> "Angular is a complete, opinionated framework — TypeScript-first, with built-in DI, routing, forms, HTTP, testing tools and SSR, all versioned together with automated migrations — which gives large teams consistency. React is a UI library focused on rendering with JSX; you compose routing, data fetching, forms and state from the ecosystem or a meta-framework like Next.js, giving flexibility and a huge community. Reactivity differs too: React re-runs components and diffs a virtual DOM, while modern Angular uses signals so only the bindings that depend on a changed value update, without a virtual DOM. I'd choose Angular for large, form-heavy enterprise apps with many teams, and React when flexibility, ecosystem or React Native matter."

**How do Angular signals compare to React hooks?**

> "`signal` is like `useState`, `computed` like a memoized derived value, and `effect` like `useEffect` — but signals track dependencies automatically, so there are no dependency arrays or stale closures, and they can live anywhere, including services, not just inside components. A signal change updates only the views that read it, whereas a React state change re-runs the component and its children unless memoized."

**What is the Angular equivalent of React Context?**

> "Dependency injection. A service provided at root is app-wide; provided in a component's `providers`, it's scoped to that subtree, much like a Context provider — but it's typed, lazily created, testable via TestBed overrides, and doesn't re-render consumers; with signals inside it, only views reading the changed signal update."

---

# 6. Key Takeaways

- Angular: framework, TS, DI, built-ins, signals, compiled templates.
- React: library, JSX, ecosystem, re-render + virtual DOM, hooks.
- Know the concept mapping table both ways.
- Choose based on team size, need for conventions, ecosystem and platform targets.

## Readiness Checklist

- [ ] Explain the comparison in 60 seconds
- [ ] Map 10 React concepts to Angular and back
- [ ] Contrast signals with hooks (tracking, closures, scope)
