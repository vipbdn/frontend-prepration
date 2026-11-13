# 11 — State Management

> **Interview + Understanding Guide** — Roadmap Section 11
>
> Goal: Classify state correctly, choose the right tool for each type, and compare Context, Redux Toolkit, Zustand, Jotai, and server-state libraries with clear trade-offs.

---

# 1. Types of State ⭐⭐⭐ (the most important idea)

Most "state management" problems come from putting the wrong kind of state in the wrong tool.

| Type | Examples | Best tool |
| --- | --- | --- |
| **Local UI state** | input value, open/closed, hover, current tab | `useState`, `useReducer` |
| **Shared client state** | theme, auth user, cart, sidebar collapsed, wizard steps | Context, Zustand, Redux Toolkit, Jotai |
| **Server state** (cache of remote data) | users, products, posts | **TanStack Query**, RTK Query, SWR, framework loaders |
| **URL state** | filters, search, pagination, selected tab, modal id | URL search params / route params |
| **Form state** | values, touched, errors, dirty | React Hook Form, Formik, `useActionState` |
| **Derived state** | totals, filtered lists | compute during render / `useMemo` / selectors |
| **Persistent state** | preferences, drafts | localStorage/IndexedDB (+ store middleware) |

### Server state is different ⭐

Server state is **owned by the server**; your app holds a **cache** of it. It can become stale, needs refetching, deduping, background updates, pagination, optimistic updates, invalidation. Storing it in Redux/Context by hand means rewriting a caching layer badly.

> Typical modern stack: **TanStack Query** for server state + **useState/Context or Zustand** for the small amount of client state + **URL** for shareable UI state. Redux Toolkit when a large client-state domain needs strict structure.

---

# 2. Principles

1. **Keep state as local as possible** (colocation) — lift only when needed.
2. **Single source of truth** for each piece of data.
3. **Don't store derived data** — compute it.
4. **Normalize** relational data (`byId` + `ids`).
5. **Put shareable UI state in the URL** (back button, deep links, refresh-safe).
6. **Separate server state from client state.**

### URL state example

```jsx
import { useSearchParams } from "react-router-dom";

function ProductFilters() {
  const [params, setParams] = useSearchParams();
  const category = params.get("category") ?? "all";
  const page = Number(params.get("page") ?? 1);

  const setCategory = (c) =>
    setParams((p) => {
      p.set("category", c);
      p.set("page", "1");
      return p;
    });

  return <CategorySelect value={category} onChange={setCategory} />;
}
```

---

# 3. Context + useReducer (built-in option)

Good for small-to-medium shared state that changes infrequently. Pitfalls: all consumers re-render; no selectors/devtools. Full guide: `10-context-api.md`.

---

# 4. Redux Toolkit (RTK) ⭐

The official, modern way to write Redux. One global store, predictable updates through actions and pure reducers, excellent devtools.

```js
const cartSlice = createSlice({
  name: "cart",
  initialState: { items: [] },
  reducers: {
    added(state, action) {
      state.items.push(action.payload); // Immer makes this immutable under the hood
    },
  },
});
```

**Strengths:** predictable, devtools/time-travel, middleware, scales to large teams, RTK Query for server state, clear conventions.
**Weaknesses:** more concepts and boilerplate than Zustand; overkill for small apps.

Full guide: `12-redux-toolkit.md`.

---

# 5. Zustand ⭐ (very popular)

Minimal store built on hooks + `useSyncExternalStore`. No provider required. Components subscribe to **slices** with selectors → only re-render when their slice changes.

```js
import { create } from "zustand";
import { persist, devtools } from "zustand/middleware";

export const useCartStore = create(
  devtools(
    persist(
      (set, get) => ({
        items: [],
        add: (product) =>
          set((state) => {
            const existing = state.items.find((i) => i.id === product.id);
            return {
              items: existing
                ? state.items.map((i) => (i.id === product.id ? { ...i, qty: i.qty + 1 } : i))
                : [...state.items, { ...product, qty: 1 }],
            };
          }),
        remove: (id) => set((state) => ({ items: state.items.filter((i) => i.id !== id) })),
        clear: () => set({ items: [] }),
        total: () => get().items.reduce((sum, i) => sum + i.price * i.qty, 0),
      }),
      { name: "cart" }, // localStorage key
    ),
  ),
);
```

```jsx
function CartBadge() {
  const count = useCartStore((s) => s.items.length); // re-renders only when count changes
  return <span>{count}</span>;
}

function AddButton({ product }) {
  const add = useCartStore((s) => s.add); // actions are stable
  return <button onClick={() => add(product)}>Add</button>;
}

// Outside React (e.g. in an API interceptor)
useCartStore.getState().clear();
```

### Selector gotcha

```jsx
// ❌ returns a new object every time → re-renders on every store change
const { items, add } = useCartStore((s) => ({ items: s.items, add: s.add }));

// ✅ select separately, or use shallow comparison
import { useShallow } from "zustand/react/shallow";
const { items, add } = useCartStore(useShallow((s) => ({ items: s.items, add: s.add })));
```

**Strengths:** tiny API, no boilerplate, selectors, works outside React, middleware (persist, devtools, immer).
**Weaknesses:** fewer conventions → large teams need discipline; global by default (SSR needs per-request stores).

---

# 6. Jotai (atomic state)

State as small independent **atoms**; components subscribe to the atoms they use. Derived atoms compose like spreadsheet cells.

```jsx
import { atom, useAtom, useAtomValue, useSetAtom } from "jotai";

const countAtom = atom(0);
const doubledAtom = atom((get) => get(countAtom) * 2); // derived

function Counter() {
  const [count, setCount] = useAtom(countAtom);
  return <button onClick={() => setCount((c) => c + 1)}>{count}</button>;
}

function Doubled() {
  return <p>{useAtomValue(doubledAtom)}</p>; // re-renders only when doubled changes
}
```

Good for fine-grained, bottom-up state (editors, complex forms, dashboards). (Recoil, a similar Meta library, is no longer maintained.)

---

# 7. Other Options (know they exist)

| Library | Model | Notes |
| --- | --- | --- |
| **MobX** | Observable, mutable, automatic dependency tracking | Very little boilerplate; "magic"; OOP-friendly |
| **Valtio** | Proxy-based mutable state | Simple; similar idea to MobX |
| **XState** | State machines / statecharts | Complex flows (checkout, wizards, media players) with explicit states |
| **Signals** (`@preact/signals-react`) | Fine-grained reactivity | Not idiomatic React |
| **Apollo Client / urql / Relay** | GraphQL server-state caches | For GraphQL APIs |

---

# 8. Comparison ⭐⭐⭐

| | Context + useReducer | Redux Toolkit | Zustand | Jotai | TanStack Query |
| --- | --- | --- | --- | --- | --- |
| For | Simple shared client state | Large client state, strict patterns | Shared client state, minimal API | Fine-grained atoms | **Server** state |
| Boilerplate | Low-medium | Medium | Very low | Very low | Low |
| Re-render control | ❌ all consumers | ✅ selectors | ✅ selectors | ✅ per atom | ✅ per query |
| Devtools | ❌ | ✅ excellent | ✅ (middleware) | ✅ | ✅ |
| Provider needed | ✅ | ✅ | ❌ | Optional | ✅ (QueryClientProvider) |
| Outside React | ❌ | ✅ | ✅ | ✅ (store API) | ✅ (queryClient) |
| Bundle | 0 | ~10–15 KB | ~1 KB | ~3 KB | ~13 KB |

### Decision guide

```text
Is it data from an API?                 → TanStack Query / RTK Query
Should it survive refresh / be shareable? → URL params (or storage)
Is it a form?                           → React Hook Form
Used by one component (or a parent + children)? → useState / useReducer, lift as needed
Shared, changes rarely (theme, auth)?   → Context
Shared, changes often / complex?        → Zustand (small/medium) or Redux Toolkit (large teams, strict structure)
Complex flow with explicit states?      → XState / useReducer with a status enum
```

---

# 9. Normalizing State

```js
// ❌ nested & duplicated
{ posts: [{ id: 1, author: { id: 7, name: "Asha" }, comments: [...] }] }

// ✅ normalized
{
  users: { byId: { 7: { id: 7, name: "Asha" } }, allIds: [7] },
  posts: { byId: { 1: { id: 1, authorId: 7, commentIds: [10] } }, allIds: [1] },
  comments: { byId: { 10: { id: 10, text: "…" } }, allIds: [10] },
}
```

Updates touch one place; lookups are O(1). RTK provides `createEntityAdapter` for this.

---

# 10. Interview Answers

**How do you decide where state lives?**

> "I first classify it. Server data goes in TanStack Query or RTK Query, which handle caching and refetching. Shareable UI state like filters and pagination goes in the URL. Form state goes in React Hook Form. Everything else starts local with `useState` and is lifted only as far as needed. For truly global client state, Context if it changes rarely, otherwise Zustand or Redux Toolkit for selectors and devtools."

**Redux vs Zustand?**

> "Both give a global store with selectors. Redux Toolkit is more structured — slices, actions, middleware, great devtools and RTK Query — which helps large teams. Zustand is a tiny hook-based store with almost no boilerplate and no provider, great for small to medium shared state. I'd choose based on team size, need for conventions, and whether we'd use RTK Query."

---

# 11. Key Takeaways

- Classify state: local, shared client, server, URL, form, derived.
- Server state ≠ client state → use a query library.
- Colocate; single source of truth; don't store derived data; normalize.
- Context for rare-change globals; Zustand/Redux/Jotai for frequent/complex shared state.
- Selector discipline prevents unnecessary re-renders.

## Readiness Checklist

- [ ] Classify 10 pieces of state from a real app into the categories
- [ ] Build the same cart with Context+reducer, RTK and Zustand
- [ ] Explain the Zustand object-selector gotcha
- [ ] Defend a state-management choice with trade-offs
