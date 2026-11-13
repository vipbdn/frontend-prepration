# 07 — Advanced Hooks 🔥

> **Interview + Understanding Guide** — Roadmap Section 7
>
> Goal: Know every built-in hook beyond the basics: `useReducer`, `useContext`, `useMemo`, `useCallback`, `useImperativeHandle`, `useSyncExternalStore`, `useTransition`, `useDeferredValue`, `useInsertionEffect`, `useDebugValue`, and the React 19 hooks (`use`, `useActionState`, `useOptimistic`, `useFormStatus`, `useEffectEvent`).

---

# 1. Hook Map

| Category | Hooks |
| --- | --- |
| State | `useState`, `useReducer` |
| Context | `useContext`, `use(Context)` |
| Refs | `useRef`, `useImperativeHandle` |
| Effects | `useEffect`, `useLayoutEffect`, `useInsertionEffect`, `useEffectEvent` |
| Performance | `useMemo`, `useCallback`, `useTransition`, `useDeferredValue` |
| External | `useSyncExternalStore` |
| Other | `useId`, `useDebugValue` |
| React 19 actions | `useActionState`, `useOptimistic`, `useFormStatus` (react-dom), `use` |

---

# 2. `useReducer` ⭐⭐⭐

```jsx
const [state, dispatch] = useReducer(reducer, initialArg, init?);
```

A **reducer** is a pure function `(state, action) => newState`. Event handlers describe **what happened** (dispatch actions); the reducer decides **how state changes**.

```jsx
const initialState = { items: [], status: "idle", error: null };

function cartReducer(state, action) {
  switch (action.type) {
    case "added": {
      const existing = state.items.find((i) => i.id === action.item.id);
      if (existing) {
        return {
          ...state,
          items: state.items.map((i) => (i.id === action.item.id ? { ...i, qty: i.qty + 1 } : i)),
        };
      }
      return { ...state, items: [...state.items, { ...action.item, qty: 1 }] };
    }
    case "removed":
      return { ...state, items: state.items.filter((i) => i.id !== action.id) };
    case "cleared":
      return initialState;
    default:
      throw new Error(`Unknown action: ${action.type}`);
  }
}

function Cart() {
  const [state, dispatch] = useReducer(cartReducer, initialState);
  return (
    <>
      <button onClick={() => dispatch({ type: "added", item: { id: 1, name: "Book" } })}>Add</button>
      <button onClick={() => dispatch({ type: "cleared" })}>Clear</button>
      <p>{state.items.length} items</p>
    </>
  );
}
```

### Lazy init

```jsx
const [state, dispatch] = useReducer(reducer, userId, (id) => loadInitialState(id));
```

### useState vs useReducer

| `useState` | `useReducer` |
| --- | --- |
| Simple, independent values | Complex state with many related fields |
| Few update types | Many update types / next state depends on several fields |
| Logic in handlers | Logic centralized and **testable** (pure reducer) |
| | Easier debugging (log actions) |
| | `dispatch` is stable → pass down without `useCallback` |

`useState` is actually implemented with a built-in reducer internally.

### Reducer rules

- Pure: no side effects, no mutation, no randomness/Date.now — React may call it twice in StrictMode.
- Actions describe **events** (`"added"`, `"task_deleted"`), not setters (`"set_items"`).
- Use **Immer** (`useImmerReducer`) for deeply nested updates.

### useReducer + Context = mini Redux

```jsx
const TasksContext = createContext(null);
const TasksDispatchContext = createContext(null);

function TasksProvider({ children }) {
  const [tasks, dispatch] = useReducer(tasksReducer, []);
  return (
    <TasksContext.Provider value={tasks}>
      <TasksDispatchContext.Provider value={dispatch}>{children}</TasksDispatchContext.Provider>
    </TasksContext.Provider>
  );
}

export const useTasks = () => useContext(TasksContext);
export const useTasksDispatch = () => useContext(TasksDispatchContext);
```

Two contexts → components that only dispatch don't re-render when tasks change. See `10-context-api.md`.

---

# 3. `useContext` ⭐⭐⭐

```jsx
const ThemeContext = createContext("light");

function Button() {
  const theme = useContext(ThemeContext); // nearest Provider above, or default
  return <button className={theme}>OK</button>;
}
```

Every consumer re-renders when the provider's `value` changes (by `Object.is`). Full guide: `10-context-api.md`.

---

# 4. `useMemo` ⭐⭐⭐

```jsx
const value = useMemo(() => computeExpensive(a, b), [a, b]);
```

Caches a **computed value** between renders until deps change.

Use for:
1. **Expensive calculations** (filtering/sorting big lists, heavy transformations)
2. **Referential stability** — objects/arrays passed to `React.memo` children, used in hook deps, or as context values

```jsx
const visible = useMemo(() => todos.filter((t) => matches(t, query)), [todos, query]);

const ctxValue = useMemo(() => ({ user, login, logout }), [user, login, logout]);
```

- It's a **performance hint**, not a semantic guarantee — React may discard the cache.
- Don't use it for side effects.
- Measuring: wrap in `console.time` — if < ~1ms, probably not worth it.

---

# 5. `useCallback` ⭐⭐⭐

```jsx
const handleSelect = useCallback((id) => setSelected(id), []);
```

Caches a **function reference**. `useCallback(fn, deps)` === `useMemo(() => fn, deps)`.

Useful only when the function identity matters:
- Passed to a `React.memo` child
- Used as a dependency of another hook (`useEffect`, `useMemo`)
- Returned from a custom hook (consumers may depend on it)

```jsx
const ProductList = memo(function ProductList({ onAdd }) { /* expensive */ });

function Shop() {
  const [cart, setCart] = useState([]);
  const handleAdd = useCallback((p) => setCart((c) => [...c, p]), []); // updater → no deps
  return <ProductList onAdd={handleAdd} />;
}
```

Pointless: `useCallback` for a handler passed to a plain `<button>` or a non-memoized child.

> With the **React Compiler**, most manual `useMemo`/`useCallback` become unnecessary. Know them anyway. Details: `15-performance-optimization.md`.

---

# 6. `useImperativeHandle`

Customizes what a parent gets when it holds a ref to your component — expose a **limited imperative API** instead of the raw DOM node.

```jsx
// React 19: ref is a regular prop for function components
function VideoPlayer({ src, ref }) {
  const videoRef = useRef(null);

  useImperativeHandle(ref, () => ({
    play: () => videoRef.current.play(),
    pause: () => videoRef.current.pause(),
    seek: (t) => { videoRef.current.currentTime = t; },
  }), []);

  return <video ref={videoRef} src={src} />;
}

function Page() {
  const playerRef = useRef(null);
  return (
    <>
      <VideoPlayer ref={playerRef} src="/intro.mp4" />
      <button onClick={() => playerRef.current.play()}>Play</button>
    </>
  );
}
```

(React ≤18: wrap the component in `forwardRef((props, ref) => …)`.)

Use sparingly — for focus, scroll, media playback, animations. Prefer props for anything declarative ("isOpen" prop, not `modal.open()`).

---

# 7. `useSyncExternalStore` ⭐

Subscribe to an **external store** (not React state) safely under concurrent rendering — avoids "tearing" (different components seeing different values during one render).

```jsx
const snapshot = useSyncExternalStore(subscribe, getSnapshot, getServerSnapshot?);
```

```jsx
// Minimal store
function createStore(initial) {
  let state = initial;
  const listeners = new Set();
  return {
    getState: () => state,
    setState: (updater) => {
      state = typeof updater === "function" ? updater(state) : updater;
      listeners.forEach((l) => l());
    },
    subscribe: (listener) => {
      listeners.add(listener);
      return () => listeners.delete(listener);
    },
  };
}

const counterStore = createStore({ count: 0 });

function useCounter(selector = (s) => s) {
  return useSyncExternalStore(counterStore.subscribe, () => selector(counterStore.getState()));
}

function Display() {
  const count = useCounter((s) => s.count);
  return <p>{count}</p>;
}
```

Rules:
- `getSnapshot` must return the **same value** if the store hasn't changed (return cached/immutable data; don't create new objects each call → infinite loop).
- `subscribe` should be stable (define outside the component or memoize).

Used internally by **Redux (react-redux), Zustand**, and for browser APIs (`navigator.onLine`, `matchMedia`).

---

# 8. `useTransition` ⭐

Mark a state update as **non-urgent** so urgent updates (typing, clicking) stay responsive. React can interrupt the transition render.

```jsx
const [isPending, startTransition] = useTransition();
```

```jsx
function TabContainer() {
  const [tab, setTab] = useState("about");
  const [isPending, startTransition] = useTransition();

  function selectTab(next) {
    startTransition(() => {
      setTab(next); // rendering a slow tab won't block clicks on other tabs
    });
  }

  return (
    <>
      <TabButtons onSelect={selectTab} />
      <div style={{ opacity: isPending ? 0.6 : 1 }}>
        {tab === "posts" ? <SlowPosts /> : <About />}
      </div>
    </>
  );
}
```

- Updates inside `startTransition` are low-priority and interruptible.
- `isPending` lets you show a subtle loading state while keeping old UI visible.
- With Suspense, transitions **keep showing old content** instead of fallback.
- React 19: `startTransition` accepts **async functions** (Actions) — `isPending` stays true until it resolves.
- ❌ Don't use for controlled input values (the input itself must update urgently).

Standalone `startTransition` (no pending state) can be imported from `react`.

---

# 9. `useDeferredValue` ⭐

Defer re-rendering a **non-urgent part of the UI** with a lagging copy of a value.

```jsx
function Search() {
  const [query, setQuery] = useState("");
  const deferredQuery = useDeferredValue(query);
  const isStale = query !== deferredQuery;

  return (
    <>
      <input value={query} onChange={(e) => setQuery(e.target.value)} /> {/* urgent */}
      <div style={{ opacity: isStale ? 0.5 : 1 }}>
        <SlowResults query={deferredQuery} /> {/* re-renders with the lagging value */}
      </div>
    </>
  );
}

const SlowResults = memo(function SlowResults({ query }) { /* expensive list */ });
```

The child must be **memoized** so it skips re-rendering while `deferredQuery` hasn't changed.

### useTransition vs useDeferredValue vs debounce

| | `useTransition` | `useDeferredValue` | Debounce |
| --- | --- | --- | --- |
| You control | The **state update** | A **value** you receive (e.g. prop) | Timing of the call |
| Mechanism | Lower render priority, interruptible | Lagging copy rendered at lower priority | Fixed delay |
| Reduces API calls? | No | No | ✅ Yes |
| Adapts to device speed? | ✅ | ✅ | ❌ fixed ms |

Use **debounce** to limit network requests; **transitions/deferred values** to keep rendering responsive.

---

# 10. `useInsertionEffect` & `useDebugValue`

```jsx
// For CSS-in-JS library authors: inject <style> before any layout effects read layout
useInsertionEffect(() => {
  injectStyle(rule);
}, [rule]);
```

```jsx
// Label custom hooks in React DevTools
function useOnlineStatus() {
  const isOnline = useSyncExternalStore(subscribe, () => navigator.onLine);
  useDebugValue(isOnline ? "Online" : "Offline");
  return isOnline;
}
```

---

# 11. React 19 Hooks

## 11.1 `use`

Reads a **Promise** or **Context** during render. Unlike other hooks, it can be called **conditionally**.

```jsx
import { use, Suspense } from "react";

function Comments({ commentsPromise }) {
  const comments = use(commentsPromise); // suspends until resolved; throws to Error Boundary if rejected
  return comments.map((c) => <p key={c.id}>{c.text}</p>);
}

function Post({ commentsPromise }) {
  return (
    <Suspense fallback={<p>Loading comments…</p>}>
      <Comments commentsPromise={commentsPromise} />
    </Suspense>
  );
}

function Heading({ show }) {
  if (!show) return null;
  const theme = use(ThemeContext); // ✅ conditional context read
  return <h1 className={theme}>Hi</h1>;
}
```

⚠️ Don't create the promise **inside** the component during render (new promise every render → suspends forever). Create it in a Server Component, loader, or cache it.

## 11.2 `useActionState`

```jsx
const [state, formAction, isPending] = useActionState(actionFn, initialState);
```

Manages state for an **Action** (async function, often a form submission). `actionFn(prevState, formData)` returns the next state.

```jsx
async function updateName(prev, formData) {
  const error = await api.updateName(formData.get("name"));
  return error ? { error } : { error: null, success: true };
}

function NameForm() {
  const [state, action, pending] = useActionState(updateName, { error: null });
  return (
    <form action={action}>
      <input name="name" />
      <button disabled={pending}>Save</button>
      {state.error && <p role="alert">{state.error}</p>}
    </form>
  );
}
```

## 11.3 `useOptimistic`

Show the **expected result immediately** while an async action is in flight; React reverts automatically when the action finishes (the real state takes over).

```jsx
function Thread({ messages, sendMessage }) {
  const [optimisticMessages, addOptimistic] = useOptimistic(
    messages,
    (current, newText) => [...current, { id: "temp", text: newText, sending: true }],
  );

  async function formAction(formData) {
    const text = formData.get("message");
    addOptimistic(text);       // show instantly
    await sendMessage(text);   // parent updates `messages` when done
  }

  return (
    <>
      {optimisticMessages.map((m) => (
        <p key={m.id} style={{ opacity: m.sending ? 0.5 : 1 }}>{m.text}</p>
      ))}
      <form action={formAction}>
        <input name="message" />
      </form>
    </>
  );
}
```

## 11.4 `useFormStatus` (from `react-dom`)

Reads the pending status of the **parent `<form>`** — for design-system submit buttons without prop drilling.

```jsx
import { useFormStatus } from "react-dom";

function SubmitButton({ children }) {
  const { pending, data, method } = useFormStatus();
  return <button disabled={pending}>{pending ? "Submitting…" : children}</button>;
}
```

Must be rendered **inside** a `<form>`.

## 11.5 `useEffectEvent`

Read latest props/state inside an effect without making them dependencies. Covered in `06-useeffect-deep-dive.md` §8.

---

# 12. Interview Answers

**When would you use useReducer over useState?**

> "When state has several related fields or many ways to change, like a cart or a multi-step form. A reducer centralizes the update logic in a pure, testable function, handlers just dispatch descriptive actions, and `dispatch` is stable so I can pass it down through context without `useCallback`."

**useMemo vs useCallback?**

> "`useMemo` caches the result of a calculation; `useCallback` caches the function itself. Both are about avoiding recomputation or keeping references stable for memoized children and dependency arrays. They have a cost, so I use them where profiling shows a benefit."

**What is useSyncExternalStore for?**

> "Subscribing to state that lives outside React — a global store or browser API — in a way that's safe with concurrent rendering, avoiding tearing. Libraries like Redux and Zustand use it under the hood."

**useTransition vs useDeferredValue?**

> "Both mark work as non-urgent. `useTransition` wraps a state update I control and gives me `isPending`. `useDeferredValue` gives me a lagging version of a value I receive, so an expensive memoized child can re-render later while the input stays responsive. Neither reduces network calls — that's what debounce is for."

---

# 13. Key Takeaways

- `useReducer`: pure reducer, descriptive actions, stable dispatch; pairs with context.
- `useMemo` (values) / `useCallback` (functions): caching for performance and referential stability.
- `useImperativeHandle`: expose a minimal imperative API via ref.
- `useSyncExternalStore`: tear-free subscriptions to external stores.
- `useTransition` / `useDeferredValue`: concurrent, interruptible, non-urgent rendering.
- React 19: `use` (promises/context, conditional), `useActionState`, `useOptimistic`, `useFormStatus`, `useEffectEvent`.

## Readiness Checklist

- [ ] Write a cart reducer + Context provider with split contexts
- [ ] Explain when useMemo/useCallback are useless
- [ ] Build a tiny store with `useSyncExternalStore`
- [ ] Demonstrate useTransition for tabs and useDeferredValue for search
- [ ] Build a form with useActionState + useFormStatus + useOptimistic
