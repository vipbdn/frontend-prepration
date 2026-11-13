# 05 — Core Hooks 🔥

> **Interview + Understanding Guide** — Roadmap Section 5
>
> Goal: Master the rules of hooks (and why they exist), `useState`, `useEffect` basics, `useRef`, `useLayoutEffect` and `useId`.
>
> Deep dives: `06-useeffect-deep-dive.md`, `07-advanced-hooks.md`, `08-custom-hooks.md`.

---

# 1. What Are Hooks?

**Hooks** are functions (starting with `use`) that let function components **use React features** — state, effects, context, refs — without classes. Introduced in **React 16.8**.

Why hooks replaced classes:
- Reuse stateful logic via **custom hooks** (instead of HOCs/render props "wrapper hell")
- Group code by **concern** instead of splitting across lifecycle methods
- No `this` binding confusion
- Smaller, easier-to-optimize components

---

# 2. Rules of Hooks ⭐⭐⭐

1. **Only call hooks at the top level** — not inside conditions, loops, nested functions, or after an early `return`.
2. **Only call hooks from React functions** — components or custom hooks (not regular functions, not class components).

```jsx
function Profile({ userId }) {
  if (!userId) return null;          // ❌ early return BEFORE hooks
  const [user, setUser] = useState(); // hook order changes between renders → bug
}

function Profile2({ show }) {
  if (show) {
    const [x] = useState(0);          // ❌ conditional hook
  }
}
```

### Why? — Hooks are stored by call order ⭐⭐⭐

React stores hook state in a **linked list on the component's fiber**. It doesn't know hooks by name — only by **position** (1st hook, 2nd hook, …).

```text
Render 1:  useState("Asha") → slot 0    useState(25) → slot 1    useEffect → slot 2
Render 2:  if a hook is skipped, slot 0's data is given to the wrong hook ❌
```

```jsx
// Simplified mental model of how React stores hooks
let hooks = [];
let index = 0;

function useState(initial) {
  const i = index++;
  if (hooks[i] === undefined) hooks[i] = initial;
  const setState = (v) => {
    hooks[i] = typeof v === "function" ? v(hooks[i]) : v;
    rerender();
  };
  return [hooks[i], setState];
}

function rerender() {
  index = 0; // reset before each render — order must match!
  render(App);
}
```

The `use()` API (React 19) is the exception — it can be called conditionally.

ESLint `eslint-plugin-react-hooks` (`rules-of-hooks`, `exhaustive-deps`) enforces these.

---

# 3. `useState` ⭐⭐⭐

```jsx
const [state, setState] = useState(initialValue);
```

### Lazy initialization

```jsx
// ❌ readFromStorage runs on EVERY render (result ignored after first)
const [todos, setTodos] = useState(readFromStorage());

// ✅ function runs only on the first render
const [todos2, setTodos2] = useState(() => readFromStorage());
```

### Functional updates

```jsx
setCount(count + 1);      // uses this render's snapshot
setCount((c) => c + 1);   // uses the latest pending value — safe for sequential/async updates
```

Use the updater form when the new state depends on the old one, especially in async callbacks, intervals and when calling the setter multiple times.

### Bail-out

If the new value is the same as the current (`Object.is`), React **may skip** re-rendering children and effects.

```jsx
setCount(5); // when count is already 5 → bail out
```

### Batching

Multiple `setState` calls in the same event (or promise, timeout — **React 18+ automatic batching**) cause **one** re-render.

```jsx
function handleClick() {
  setA(1);
  setB(2);
  setC(3); // → 1 render
}
```

### Setter is stable

`setState` has a **stable identity** — safe to omit from dependency arrays.

### Storing functions in state

```jsx
const [fn, setFn] = useState(() => () => console.log("hi")); // initializer returns the fn
setFn(() => newFn); // updater returns the fn
```

---

# 4. `useEffect` (basics) ⭐⭐⭐

Runs **side effects after the commit** — synchronizing with something outside React (network, DOM APIs, subscriptions, timers).

```jsx
useEffect(() => {
  // setup
  return () => {
    // cleanup (before next run and on unmount)
  };
}, [dependencies]);
```

| Dependency array | Runs |
| --- | --- |
| omitted | after **every** render |
| `[]` | after the **first** render (mount), cleanup on unmount |
| `[a, b]` | after mount and whenever `a` or `b` changed (`Object.is`) |

```jsx
useEffect(() => {
  const id = setInterval(() => setSeconds((s) => s + 1), 1000);
  return () => clearInterval(id);
}, []);

useEffect(() => {
  document.title = `${count} new messages`;
}, [count]);
```

- Effects run **after paint** (non-blocking) in most cases.
- In StrictMode dev, mount effects run **setup → cleanup → setup**.

Everything else about effects → `06-useeffect-deep-dive.md`.

---

# 5. `useRef` ⭐⭐⭐

```jsx
const ref = useRef(initialValue); // { current: initialValue }
```

A ref is a **mutable box that persists across renders** and **does not trigger re-renders** when changed.

### Use 1: DOM access

```jsx
function SearchBox() {
  const inputRef = useRef(null);
  useEffect(() => {
    inputRef.current.focus();
  }, []);
  return <input ref={inputRef} />;
}
```

### Use 2: Instance variables (values that shouldn't cause renders)

```jsx
function Stopwatch() {
  const [time, setTime] = useState(0);
  const intervalRef = useRef(null);

  const start = () => {
    if (intervalRef.current) return;
    intervalRef.current = setInterval(() => setTime((t) => t + 1), 1000);
  };
  const stop = () => {
    clearInterval(intervalRef.current);
    intervalRef.current = null;
  };

  return (
    <>
      <p>{time}s</p>
      <button onClick={start}>Start</button>
      <button onClick={stop}>Stop</button>
    </>
  );
}
```

Other uses: previous values, latest callback (avoid stale closures), "is mounted" flags, render counters, storing a debounced function instance.

### useRef vs useState

| | `useRef` | `useState` |
| --- | --- | --- |
| Persists across renders | ✅ | ✅ |
| Triggers re-render on change | ❌ | ✅ |
| Mutable | ✅ `ref.current = x` | ❌ use setter |
| Read during render | ❌ avoid (except lazy init) | ✅ |

> **Don't read or write `ref.current` during render** (except initializing). It makes rendering impure and unpredictable. Read/write refs in **effects and event handlers**.

### Lazy ref initialization

```jsx
const playerRef = useRef(null);
if (playerRef.current === null) {
  playerRef.current = new VideoPlayer(); // created once — allowed pattern
}
```

More on refs (callback refs, forwarding, imperative handle): `17-refs-portals-dom.md`.

---

# 6. `useLayoutEffect` ⭐

Same API as `useEffect`, but runs **synchronously after DOM mutations and before the browser paints**.

```text
render → commit (DOM updated) → useLayoutEffect → PAINT → useEffect
```

Use it when you must **measure the DOM and update before the user sees anything** (avoid flicker):

```jsx
function Tooltip({ targetRect, children }) {
  const ref = useRef(null);
  const [height, setHeight] = useState(0);

  useLayoutEffect(() => {
    setHeight(ref.current.getBoundingClientRect().height); // measured before paint
  }, []);

  const top = targetRect.top - height; // place above target; no visible jump
  return <div ref={ref} style={{ position: "absolute", top }}>{children}</div>;
}
```

| `useEffect` | `useLayoutEffect` |
| --- | --- |
| After paint (async) | Before paint (sync) |
| Doesn't block visual updates | Blocks paint → can hurt performance |
| Data fetching, subscriptions, logging | DOM measurements, scroll position, preventing flicker |
| Default choice | Only when you see flicker |

- On the server, `useLayoutEffect` doesn't run (warning in older versions) — use `useEffect` or render client-only.
- `useInsertionEffect` runs even earlier (for CSS-in-JS libraries injecting styles).

---

# 7. `useId`

Generates a **unique, stable ID** that matches between server and client (hydration-safe). For accessibility attributes, not list keys.

```jsx
function Field({ label }) {
  const id = useId();
  return (
    <>
      <label htmlFor={id}>{label}</label>
      <input id={id} aria-describedby={`${id}-hint`} />
      <p id={`${id}-hint`}>We'll never share it.</p>
    </>
  );
}
```

Why not `Math.random()` or a counter? They differ between server and client → **hydration mismatch**, and change across renders.

---

# 8. Output Questions 🧪

### Q1

```jsx
function App() {
  const [n, setN] = useState(0);
  const ref = useRef(0);
  return (
    <button onClick={() => { ref.current++; }}>
      {ref.current}
    </button>
  );
}
```
Click 3 times. What's displayed?

<details><summary>Answer</summary>`0` — mutating a ref doesn't re-render. The next unrelated re-render would show `3`.</details>

### Q2

```jsx
function App() {
  const [count, setCount] = useState(0);
  useEffect(() => {
    console.log("effect", count);
  });
  console.log("render", count);
  return <button onClick={() => setCount(1)}>{count}</button>;
}
```
Logs on mount and after clicking twice (production, no StrictMode)?

<details><summary>Answer</summary>

Mount: `render 0`, `effect 0`. First click: `render 1`, `effect 1`. Second click: `setCount(1)` with same value → React may render the component once more without committing (`render 1`) and then bails out — no effect. (React can do one extra render before bailing; don't rely on exact counts.)

</details>

### Q3

```jsx
function App() {
  const [items] = useState(() => {
    console.log("init");
    return [];
  });
  const [n, setN] = useState(0);
  return <button onClick={() => setN(n + 1)}>{n}</button>;
}
```
How many times does `init` log after 5 clicks (production)?

<details><summary>Answer</summary>Once — lazy initializer runs only on mount.</details>

---

# 9. Interview Answers

**Why can't hooks be called conditionally?**

> "React identifies hooks by their call order, storing their state in a list on the component's fiber. If a hook is skipped in one render, every hook after it reads the wrong state. Calling hooks unconditionally at the top level keeps the order identical every render."

**useRef vs useState?**

> "Both persist across renders. Updating state triggers a re-render and should be used for anything shown in the UI. A ref is a mutable container that doesn't trigger re-renders — for DOM nodes, timer IDs, previous values or the latest callback. I avoid reading refs during render."

**useEffect vs useLayoutEffect?**

> "`useEffect` runs after the browser paints, so it doesn't block the UI — the default for data fetching and subscriptions. `useLayoutEffect` runs synchronously after DOM updates but before paint, so I use it only for measuring layout and adjusting before the user sees a flicker."

---

# 10. Key Takeaways

- Hooks: top level only, React functions only — because state is stored by call order.
- `useState`: lazy init, functional updates, stable setter, bail-out on same value, automatic batching.
- `useEffect`: post-commit side effects with deps + cleanup.
- `useRef`: mutable, persistent, no re-render; DOM refs and instance values; don't touch during render.
- `useLayoutEffect`: before paint, for measurements; use sparingly.
- `useId`: SSR-safe IDs for a11y attributes.

## Readiness Checklist

- [ ] Implement a toy `useState` showing why order matters
- [ ] Explain lazy initialization and functional updates
- [ ] Build a stopwatch using `useRef` for the interval id
- [ ] Explain when `useLayoutEffect` is required
- [ ] Explain why `useId` exists
