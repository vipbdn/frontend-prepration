# 15 — Performance Optimization 🔥🔥🔥

> **Interview + Understanding Guide** — Roadmap Section 15
>
> Goal: Diagnose and fix React performance problems systematically: measure first, reduce unnecessary renders structurally, memoize where it pays off, split code, virtualize long lists, and know what the React Compiler changes.

---

# 1. The Process

```text
1. MEASURE   React DevTools Profiler, Performance panel, Web Vitals (INP, LCP, CLS)
2. IDENTIFY  Which components render too often / take too long? Large bundle? Long tasks?
3. FIX       Structure first (colocation, composition) → memoization → splitting/virtualization
4. VERIFY    Measure again
```

> "Premature optimization is the root of all evil." Don't sprinkle `useMemo` everywhere — profile.

### Two kinds of React performance problems

| Problem | Symptoms | Fixes |
| --- | --- | --- |
| **Runtime** (rendering) | Typing lag, janky interactions, slow updates (bad INP) | Fewer/cheaper renders, memoization, virtualization, transitions, Web Workers |
| **Load** (bundle/network) | Slow first load (bad LCP), big JS | Code splitting, lazy loading, tree shaking, SSR/RSC, image optimization, caching |

---

# 2. Measuring

### React DevTools Profiler

1. Record → interact → stop.
2. **Flamegraph**: which components rendered and how long.
3. **Ranked**: slowest components.
4. Enable *"Record why each component rendered"* → shows "props changed: onClick", "hook 2 changed", "parent rendered".
5. Settings → *"Highlight updates when components render"* to see re-renders live.

### `<Profiler>` API

```jsx
<Profiler id="ProductGrid" onRender={(id, phase, actualDuration, baseDuration) => {
  if (actualDuration > 16) console.warn(id, phase, actualDuration);
}}>
  <ProductGrid />
</Profiler>
```

### Chrome Performance panel

Long tasks (> 50 ms), scripting vs rendering time, layout thrashing. Test with **CPU throttling** (4–6×) to simulate real devices. Always profile a **production build** (dev mode is much slower).

---

# 3. Structural Fixes First ⭐⭐⭐ (no memo needed)

### 3.1 Move state down (colocation)

```jsx
// ❌ Typing re-renders the whole page including ExpensiveTree
function Page() {
  const [query, setQuery] = useState("");
  return (
    <>
      <input value={query} onChange={(e) => setQuery(e.target.value)} />
      <ExpensiveTree />
    </>
  );
}

// ✅ Isolate the state in its own component
function SearchInput() {
  const [query, setQuery] = useState("");
  return <input value={query} onChange={(e) => setQuery(e.target.value)} />;
}
function Page() {
  return (
    <>
      <SearchInput />
      <ExpensiveTree /> {/* no longer re-renders on typing */}
    </>
  );
}
```

### 3.2 Lift content up (children as props)

When the stateful part **wraps** the expensive part:

```jsx
// ❌
function ColorPicker() {
  const [color, setColor] = useState("red");
  return (
    <div style={{ color }}>
      <input value={color} onChange={(e) => setColor(e.target.value)} />
      <ExpensiveTree />
    </div>
  );
}

// ✅ ExpensiveTree is created by the parent → same element reference → skipped
function ColorPicker({ children }) {
  const [color, setColor] = useState("red");
  return (
    <div style={{ color }}>
      <input value={color} onChange={(e) => setColor(e.target.value)} />
      {children}
    </div>
  );
}

<ColorPicker>
  <ExpensiveTree />
</ColorPicker>
```

### 3.3 Split contexts / avoid broad context values

See `10-context-api.md`.

### 3.4 Avoid unnecessary effects

Effects that set state cause **extra render passes**. Derive values during render instead (`06-useeffect-deep-dive.md` §6).

### 3.5 Stable keys

Random keys remount items every render; index keys break reorders.

### 3.6 Don't define components inside components

Causes remounts on every render (lost state, expensive DOM recreation).

---

# 4. Memoization ⭐⭐⭐

### `React.memo`

```jsx
const ProductCard = memo(function ProductCard({ product, onAdd }) {
  return (
    <div>
      <h3>{product.name}</h3>
      <button onClick={() => onAdd(product.id)}>Add</button>
    </div>
  );
});
```

Skips re-rendering when props are **shallowly equal** (`Object.is` per prop). Custom comparator: `memo(Comp, (prev, next) => prev.id === next.id)` — return `true` to **skip**.

### Making memo actually work

`memo` is useless if any prop changes reference every render:

```jsx
function Shop({ products }) {
  const [cart, setCart] = useState([]);

  // ❌ new function / object / array each render → memo always fails
  // <ProductCard onAdd={(id) => …} style={{ margin: 8 }} tags={["new"]} />

  const handleAdd = useCallback((id) => setCart((c) => [...c, id]), []);
  return products.map((p) => <ProductCard key={p.id} product={p} onAdd={handleAdd} />);
}
```

Checklist for memoized children:
- Functions → `useCallback` (or functional state updates to keep deps empty)
- Objects/arrays → `useMemo` or hoist outside the component
- `children` JSX → new element every render (breaks memo unless memoized)
- Context values used inside still trigger re-renders

### `useMemo` for expensive work

```jsx
const sorted = useMemo(() => [...items].sort(compareFn), [items, compareFn]);
```

Measure with `console.time` — if it's under ~1 ms, skip `useMemo`.

### When memoization hurts

- Extra memory + comparison cost on every render
- Dependency array bugs (stale values)
- Readability
- Useless when props change every render anyway

---

# 5. React Compiler ⭐ (modern)

The **React Compiler** (stable since late 2025, works with React 17+ and is designed for React 19) is a **build-time** Babel/SWC plugin that **automatically memoizes** components, hooks, values and JSX — like inserting `useMemo`/`useCallback`/`memo` everywhere it's safe, at a finer granularity than hand-written code.

```jsx
// You write:
function ProductList({ products, filter }) {
  const visible = products.filter((p) => p.category === filter);
  const handleClick = (id) => track(id);
  return visible.map((p) => <Row key={p.id} product={p} onClick={handleClick} />);
}
// The compiler caches `visible`, `handleClick` and the JSX between renders when inputs are unchanged.
```

- Requires code that follows the **Rules of React** (pure render, no mutation of props/state, rules of hooks). The ESLint plugin flags violations; the compiler skips components it can't prove safe.
- Reduces the need for manual memoization, but you should still understand it (existing codebases, interviews, edge cases).

---

# 6. Code Splitting & Lazy Loading ⭐

```jsx
import { lazy, Suspense } from "react";

const AdminDashboard = lazy(() => import("./AdminDashboard"));
const RichEditor = lazy(() => import("./RichEditor"));

function App() {
  return (
    <Suspense fallback={<PageSpinner />}>
      <Routes>
        <Route path="/admin" element={<AdminDashboard />} />
      </Routes>
    </Suspense>
  );
}
```

Strategies:
- **Route-based** splitting (biggest win)
- **Interaction-based**: modals, editors, charts loaded on demand
- **Below-the-fold** components
- **Prefetch** on hover/idle: `const load = () => import("./Editor")` called in `onMouseEnter`
- Heavy libraries: dynamic `import()` inside handlers (`const { jsPDF } = await import("jspdf")`)

`React.lazy` requires a **default export** (or `.then(m => ({ default: m.Named }))`).

Bundle analysis: `rollup-plugin-visualizer` (Vite), `webpack-bundle-analyzer`, `source-map-explorer`.

---

# 7. Virtualization (Windowing) ⭐

Render only the visible rows of a long list.

```jsx
import { useVirtualizer } from "@tanstack/react-virtual";

function VirtualList({ items }) {
  const parentRef = useRef(null);
  const virtualizer = useVirtualizer({
    count: items.length,
    getScrollElement: () => parentRef.current,
    estimateSize: () => 48,
    overscan: 8,
  });

  return (
    <div ref={parentRef} style={{ height: 600, overflow: "auto" }}>
      <div style={{ height: virtualizer.getTotalSize(), position: "relative" }}>
        {virtualizer.getVirtualItems().map((row) => (
          <div
            key={row.key}
            style={{ position: "absolute", top: 0, left: 0, width: "100%", height: row.size,
                     transform: `translateY(${row.start}px)` }}
          >
            {items[row.index].name}
          </div>
        ))}
      </div>
    </div>
  );
}
```

Libraries: TanStack Virtual, react-window, react-virtuoso (dynamic heights, chat). Use for 100s–1000s+ rows/cards. The concept from scratch: `32-machine-coding.md`.

---

# 8. Concurrent Features for Responsiveness

```jsx
const [isPending, startTransition] = useTransition();
startTransition(() => setFilter(next)); // heavy re-render won't block typing/clicks

const deferredQuery = useDeferredValue(query); // render expensive results with lagging value
```

See `19-concurrent-react.md`.

---

# 9. Other Runtime Techniques

| Technique | Use |
| --- | --- |
| Debounce / throttle | Limit work on typing, scroll, resize |
| `useRef` for non-visual values | Avoid renders for values not shown in UI |
| Batch state updates | One reducer dispatch instead of 5 setters (already batched, but fewer state pieces = simpler) |
| Keep state normalized & flat | Smaller updates, fewer cascades |
| Web Workers | Move CPU-heavy work (parsing, sorting 100k rows) off the main thread |
| CSS for animations | `transform`/`opacity`, not state-driven per-frame renders |
| Avoid inline heavy computations in JSX | Compute once per render or memoize |
| Selectors in stores | Subscribe to slices (Redux, Zustand) |

---

# 10. Load Performance

- Code splitting + lazy routes
- **Tree-shakeable imports** (`lodash-es`, `date-fns`), remove heavy deps (moment)
- **Images**: modern formats (AVIF/WebP), `srcset`/`sizes`, `loading="lazy"`, explicit dimensions (CLS), priority for LCP image (`fetchpriority="high"`, Next `<Image priority>`)
- **Fonts**: `font-display: swap`, preload, subset
- **SSR/SSG/RSC** to show content before JS loads (`21-ssr-hydration-server-components.md`)
- HTTP caching with hashed filenames, CDN, Brotli compression
- Preload critical resources, prefetch next routes

### Web Vitals to quote

| Metric | Measures | Good |
| --- | --- | --- |
| LCP | Loading of main content | ≤ 2.5 s |
| INP | Responsiveness (interaction → next paint) | ≤ 200 ms |
| CLS | Layout stability | ≤ 0.1 |

---

# 11. Common Interview Scenario

> "A list page with 5,000 items and a search box is laggy when typing. How do you fix it?"

1. **Profile** — confirm the list re-render on every keystroke is the cost.
2. **Colocate** input state or pass the list via `children` so typing doesn't re-render unrelated parts.
3. **Virtualize** the list → render ~30 rows instead of 5,000.
4. **Memoize** filtering with `useMemo([items, query])` and row components with `memo` + stable callbacks.
5. **`useDeferredValue(query)`** so the input stays responsive while results update at lower priority.
6. **Debounce** if filtering hits the server.
7. Move heavy filtering to a **Web Worker** if still slow.
8. Re-measure.

---

# 12. Interview Answers

**How do you optimize a slow React app?**

> "Measure first with the React Profiler and the Performance panel on a production build. For rendering issues, I fix structure first — move state down, pass expensive subtrees as children, split contexts — then memoize where profiling shows benefit with `memo`, `useMemo` and `useCallback`, virtualize long lists, and use transitions or deferred values to keep input responsive. For load time: route-based code splitting, lazy loading heavy components, trimming dependencies, image optimization and SSR where needed."

**When does React.memo not help?**

> "When props change reference every render — inline functions, objects, arrays or JSX children — or when the component re-renders due to its own state or a context change. It also adds comparison cost, so it's not worth it for cheap components."

**What does the React Compiler do?**

> "It's a build-time plugin that automatically memoizes components, values and callbacks where it can prove it's safe under the Rules of React. It removes most of the need for manual `useMemo`, `useCallback` and `memo`, but the code has to be pure and follow hook rules."

---

# 13. Key Takeaways

- Measure (Profiler, prod build, CPU throttling) before optimizing.
- Structure first: colocate state, lift content up, split contexts, avoid needless effects.
- `memo` needs stable props (`useCallback`, `useMemo`, hoisting).
- React Compiler automates memoization for pure code.
- Code split routes and heavy components; virtualize long lists.
- Transitions/deferred values for responsiveness; debounce for network.
- Track LCP, INP, CLS.

## Readiness Checklist

- [ ] Profile and explain a "why did this render" result
- [ ] Demonstrate move-state-down and children-as-props fixes
- [ ] Make a broken `memo` work by stabilizing props
- [ ] Lazy-load routes and a modal with prefetch on hover
- [ ] Virtualize a 10k-row list
- [ ] Walk through the laggy-search scenario end to end
