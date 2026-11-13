# 09 — Rendering, Reconciliation & Fiber 🔥🔥🔥

> **Interview + Understanding Guide** — Roadmap Section 9
>
> Goal: Explain precisely **when and why components re-render**, how reconciliation and diffing work, what Fiber is, and how batching and bail-outs behave. This is the #1 internals topic for 4-YOE interviews.

---

# 1. What Triggers a Re-render? ⭐⭐⭐

A component re-renders when:

1. **Its state changes** (`setState` / `dispatch` with a different value)
2. **Its parent re-renders** (by default, all children re-render — regardless of props!)
3. **A context it consumes changes** (`useContext` value changes)
4. (A hook it uses triggers 1 or 3 — e.g. a store subscription via `useSyncExternalStore`)

### ❌ Myth: "A component re-renders when its props change."

Props changing is just a consequence of the parent re-rendering. If the parent re-renders, the child re-renders **even with identical props** — unless the child is wrapped in `React.memo`.

```jsx
function Parent() {
  const [count, setCount] = useState(0);
  return (
    <>
      <button onClick={() => setCount(count + 1)}>{count}</button>
      <StaticChild /> {/* re-renders on every click, even with no props */}
    </>
  );
}
```

### Re-render ≠ DOM update

A re-render calls the function and diffs output. If nothing changed, **no DOM work** happens. Re-renders are usually cheap; they matter when components are expensive or very numerous.

### Setting state to the same value

```jsx
setCount(count); // Object.is equal → React bails out (may render the component once more without committing children)
```

---

# 2. Render Phase vs Commit Phase ⭐⭐⭐

```text
┌──────────────── RENDER PHASE ────────────────┐   ┌──────────── COMMIT PHASE ────────────┐
│ Call components, compute new element tree     │   │ Apply DOM mutations                   │
│ Reconcile (diff) with current fiber tree      │ → │ Update refs                           │
│ Pure — no side effects                        │   │ Run useLayoutEffect (sync, pre-paint) │
│ Can be paused, interrupted, restarted (concurrent)│ │ Browser paints                        │
│ Can run more than once (StrictMode)           │   │ Run useEffect (after paint)           │
└───────────────────────────────────────────────┘   └───────────────────────────────────────┘
```

- Render phase: **interruptible** in concurrent mode.
- Commit phase: **synchronous**, can't be interrupted (the DOM must be consistent).

This is why render must be pure: React may call it, discard the result, and call again.

---

# 3. Reconciliation ⭐⭐⭐

**Reconciliation** is the process of comparing the new element tree with the previous one to determine the minimal set of changes.

A general tree diff is O(n³). React uses an **O(n) heuristic** based on two assumptions:

1. **Elements of different types produce different trees.**
2. **Keys** identify which children are stable across renders.

### Rule 1: Different type → tear down and rebuild

```jsx
// before          // after
<div>               <section>
  <Counter />         <Counter />
</div>              </section>
```

`div` → `section`: React **unmounts** the whole subtree (Counter loses state, effects clean up, DOM nodes destroyed) and mounts a new one.

Same for components: `<UserCard />` → `<AdminCard />` at the same position → full remount.

### Rule 2: Same DOM type → keep node, update attributes

```jsx
<div className="before" title="x" />
<div className="after" title="x" />
// → only className is updated on the existing DOM node
```

### Rule 3: Same component type → keep instance, re-render with new props

State is preserved; the component function runs again; React recurses into its children.

### Rule 4: Children lists → keys

Without keys, React compares children **by index**. Inserting at the start makes every item look "changed". With keys, React matches by identity and only inserts/moves what's needed. (Details and the index-key bug: `02-jsx-conditional-rendering-lists.md` §5.)

```jsx
<ul>
  {items.map((item) => <li key={item.id}>{item.name}</li>)}
</ul>
```

### Position matters (not where JSX is written)

```jsx
// Both branches render <Counter /> at the SAME position → state preserved when toggling
{isFancy ? <Counter isFancy /> : <Counter />}

// Different positions → state reset
{isFancy && <Counter isFancy />}
{!isFancy && <Counter />}
```

In the second example, the two `&&` expressions occupy **two different child slots** (one is `false` while the other is a Counter), so toggling unmounts one and mounts the other.

---

# 4. Fiber Architecture ⭐

**Fiber** (React 16+) is the reimplementation of React's reconciler. A **fiber** is a JavaScript object representing **a unit of work** — one per component/element in the tree.

### Why Fiber?

The old "stack reconciler" processed the whole tree **recursively and synchronously** — a large update blocked the main thread (janky input, dropped frames). Fiber makes rendering **incremental**:

- **Split** rendering into small units of work
- **Pause**, **resume**, **abort** work
- Assign **priorities** (typing > data list update)
- **Reuse** previously completed work
- Enables concurrent features: transitions, Suspense, streaming SSR

### Fiber node (simplified)

```js
{
  type: UserCard,          // function, class, or "div"
  key: "42",
  stateNode: domNodeOrInstance,
  child: firstChildFiber,  // linked-list tree:
  sibling: nextSiblingFiber,
  return: parentFiber,
  pendingProps, memoizedProps,
  memoizedState,           // hooks linked list for function components
  updateQueue,
  flags,                   // side effects to apply in commit (Placement, Update, Deletion)
  lanes,                   // priority
  alternate,               // the counterpart fiber in the other tree
}
```

### Double buffering

React keeps **two trees**:
- **current** — what's on screen
- **work-in-progress** — being built during render

On commit, React swaps the pointer: work-in-progress becomes current. Each fiber's `alternate` points to its twin, so fibers are reused instead of re-allocated.

### Work loop (conceptual)

```js
function workLoopConcurrent() {
  while (workInProgress !== null && !shouldYield()) {
    workInProgress = performUnitOfWork(workInProgress); // beginWork → children → completeWork
  }
}
// shouldYield(): ~5ms time slice elapsed? Let the browser handle input/paint, resume later.
```

The **Scheduler** package decides what to run and when. **Lanes** (bitmask priorities) represent update priorities: SyncLane (discrete input like clicks), InputContinuousLane, DefaultLane, TransitionLanes, IdleLane…

---

# 5. Batching ⭐

**Batching** groups multiple state updates into a single re-render.

```jsx
function handleClick() {
  setCount((c) => c + 1);
  setFlag((f) => !f);
  setName("x");
  // → ONE render
}
```

### Automatic batching (React 18+) ⭐

| Context | React 17 (legacy `ReactDOM.render`) | React 18+ (`createRoot`) |
| --- | --- | --- |
| React event handlers | Batched | Batched |
| `setTimeout`, promises, native listeners | ❌ Each update renders separately | ✅ Batched |

```jsx
fetchData().then(() => {
  setLoading(false);
  setData(result); // React 18: one render; React 17: two renders
});
```

### Opting out: `flushSync`

```jsx
import { flushSync } from "react-dom";

flushSync(() => setItems([...items, newItem])); // DOM updated synchronously here
listRef.current.lastChild.scrollIntoView();      // can measure the new DOM immediately
```

Rarely needed; hurts performance.

### Update queue processing

```jsx
setN(n + 5);          // replace with 5
setN((p) => p + 1);   // 5 → 6
setN(42);             // replace with 42
// final: 42
```

React processes queued updates in order during the next render.

---

# 6. Bail-outs: When React Skips Work

| Mechanism | Skips |
| --- | --- |
| `setState(sameValue)` (`Object.is`) | Re-rendering children & effects |
| `React.memo(Component)` | Re-render if props are shallowly equal |
| Same element reference (`children` prop, `useMemo`'d JSX) | Re-render of that subtree |
| `PureComponent` / `shouldComponentUpdate` (classes) | Re-render |

### The "same element reference" bail-out ⭐

If a component returns the **exact same element object** as last time, React skips reconciling it.

```jsx
function ScrollTracker({ children }) {
  const [y, setY] = useState(0);
  useEffect(() => {
    const onScroll = () => setY(window.scrollY);
    window.addEventListener("scroll", onScroll);
    return () => window.removeEventListener("scroll", onScroll);
  }, []);
  return (
    <div>
      <p>{y}</p>
      {children} {/* created by the PARENT; same reference on ScrollTracker's re-renders → not re-rendered */}
    </div>
  );
}

<ScrollTracker>
  <ExpensiveTree />
</ScrollTracker>
```

This is the basis of "lift content up" / "move state down" optimization (`15-performance-optimization.md`).

---

# 7. Context and Re-renders

When a provider's `value` changes, **all consumers** re-render — even if wrapped in `React.memo`, and even if they only use part of the value.

```jsx
<AuthContext.Provider value={{ user, login }}> {/* new object every render → all consumers re-render */}
```

Fixes: memoize the value, split contexts, selectors via external stores. See `10-context-api.md`.

---

# 8. Debugging Re-renders

- **React DevTools → Profiler**: record, see which components rendered, how long, and **"Why did this render?"** (enable in settings).
- **"Highlight updates when components render"** in DevTools settings.
- `console.log("render", name)` in the component body (remember StrictMode doubles it in dev).
- `why-did-you-render` library.
- `<Profiler id onRender>` API for programmatic measurement.

```jsx
import { Profiler } from "react";
<Profiler id="Feed" onRender={(id, phase, actualDuration) => console.log(id, phase, actualDuration)}>
  <Feed />
</Profiler>
```

---

# 9. Output Questions 🧪

### Q1

```jsx
function Child() { console.log("child"); return null; }
function Parent() {
  const [n, setN] = useState(0);
  return <><Child /><button onClick={() => setN(n + 1)} /></>;
}
```
Click 3 times — how many times does `child` log (prod)?

<details><summary>Answer</summary>`4` — 1 mount + 3 re-renders (Child isn't memoized).</details>

### Q2

```jsx
const Child = memo(function Child({ onClick }) { console.log("child"); return null; });
function Parent() {
  const [n, setN] = useState(0);
  return <><Child onClick={() => {}} /><button onClick={() => setN(n + 1)} /></>;
}
```
Click 3 times?

<details><summary>Answer</summary>`4` — the inline arrow is a new function each render, so memo's shallow comparison fails.</details>

### Q3

```jsx
function App() {
  const [show, setShow] = useState(true);
  return (
    <div>
      {show && <Counter />}
      <Counter />
    </div>
  );
}
```
Second Counter shows 5. Toggle `show` to false. Value of the remaining Counter?

<details><summary>Answer</summary>`5` — the `{show && …}` slot becomes `false` but still occupies position 0; the second Counter stays at position 1 and keeps state.</details>

### Q4

```jsx
function App() {
  const [type, setType] = useState("div");
  const Wrapper = type;
  return <Wrapper><input /></Wrapper>;
}
```
Type into the input, then switch `type` to `"section"`. What happens to the text?

<details><summary>Answer</summary>Lost — different element type at the root → whole subtree unmounted and remounted.</details>

### Q5

```jsx
function App() {
  const [a, setA] = useState(0);
  const [b, setB] = useState(0);
  console.log("render");
  return <button onClick={() => setTimeout(() => { setA(1); setB(1); })}>go</button>;
}
```
Renders after click in React 18 with `createRoot`?

<details><summary>Answer</summary>One extra render (automatic batching inside `setTimeout`). In React 17 with `ReactDOM.render`: two.</details>

---

# 10. Interview Answers

**When does a React component re-render?**

> "When its state changes, when its parent re-renders, or when a context it consumes changes. A common misconception is that props changes cause re-renders — actually a parent re-render re-renders all children by default, whether or not props changed, unless the child is memoized. And a re-render doesn't necessarily mean DOM updates — React only commits the differences."

**Explain reconciliation.**

> "After a render, React compares the new element tree with the previous one. It uses heuristics to make this O(n): elements of a different type mean the subtree is destroyed and rebuilt; same DOM type means update the attributes; same component type means keep the instance and state and re-render with new props; and for lists it uses keys to match children. Then the commit phase applies the minimal DOM changes."

**What is Fiber?**

> "Fiber is React's reconciliation engine since v16. Each element becomes a fiber node — a unit of work in a linked-list tree. That lets React split rendering into small chunks, pause to let the browser handle input, prioritize urgent updates, and resume or discard work. It uses double buffering: a current tree and a work-in-progress tree swapped at commit. It's the foundation for concurrent features like transitions and Suspense."

**What is automatic batching?**

> "Since React 18 with `createRoot`, multiple state updates are batched into one render no matter where they happen — event handlers, promises, timeouts or native listeners. Before, only React event handlers were batched. `flushSync` opts out when you need the DOM updated immediately."

---

# 11. Key Takeaways

- Re-render triggers: own state, parent re-render, context change. Not "props changed".
- Re-render ≠ DOM change; commit applies only diffs.
- Render (pure, interruptible) → Commit (sync: DOM, refs, layout effects, paint, effects).
- Reconciliation: different type → remount; same type → update; lists → keys; position matters.
- Fiber: units of work, linked tree, priorities (lanes), time-slicing, double buffering.
- React 18 automatic batching everywhere; `flushSync` to opt out.
- Bail-outs: same state, `memo`, same element reference.

## Readiness Checklist

- [ ] Explain the 3 re-render triggers and bust the props myth
- [ ] Draw render vs commit phases with effects timing
- [ ] Explain the 4 reconciliation rules with examples
- [ ] Explain Fiber, double buffering, lanes in 2 minutes
- [ ] Explain automatic batching and flushSync
- [ ] Use the Profiler to find an unnecessary re-render
