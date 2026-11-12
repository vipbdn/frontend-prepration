# 26 — React + Advanced JavaScript 🔥🔥🔥

> **Interview + Understanding Guide** — `00-starter-roadmap.md` Section 26
>
> Goal: Connect every advanced JS concept to React — closures and stale closures, references and immutability, memoization, async data fetching with race conditions and cancellation, cleanup, and debounce/throttle. This is where React interviews test your JavaScript.

---

# 1. The Mental Model: Every Render is a Snapshot ⭐⭐⭐

A function component is **just a function**. Every render **calls it again**, creating:
- new local variables
- new functions (event handlers, callbacks)
- new objects/arrays

Each render's functions **close over that render's** props and state.

```jsx
function Counter() {
  const [count, setCount] = useState(0);

  function handleClick() {
    // `count` is the value from THE RENDER this handler was created in
    setCount(count + 1);
  }

  return <button onClick={handleClick}>{count}</button>;
}
```

```text
Render 1: count = 0 → handleClick₁ closes over count = 0
Render 2: count = 1 → handleClick₂ closes over count = 1
```

> State isn't a variable that changes — each render **has its own constant** `count`. Updating state schedules a **new render** with a new value.

---

# 2. Closures in React ⭐⭐⭐

## 2.1 Event handlers + closures

```jsx
function Counter() {
  const [count, setCount] = useState(0);

  const handleTripleClick = () => {
    setCount(count + 1); // count = 0 → 1
    setCount(count + 1); // count still 0 → 1
    setCount(count + 1); // count still 0 → 1
  };
  // Result: 1, not 3 ❗

  const handleTripleFixed = () => {
    setCount((c) => c + 1); // functional update: React passes the latest queued value
    setCount((c) => c + 1);
    setCount((c) => c + 1);
  };
  // Result: 3 ✅
}
```

**Rule:** when the next state depends on the previous state, use the **functional updater** `setX(prev => ...)`.

## 2.2 Stale closures ⭐⭐⭐

A **stale closure** is a function that captured an **old render's** values and is still being called later.

```jsx
function Timer() {
  const [count, setCount] = useState(0);

  useEffect(() => {
    const id = setInterval(() => {
      console.log(count);   // always 0 ❗
      setCount(count + 1);  // always sets 1 ❗
    }, 1000);
    return () => clearInterval(id);
  }, []); // effect runs once → interval callback closes over render 1's count = 0

  return <p>{count}</p>; // stuck at 1
}
```

### Fixes

```jsx
// ✅ Fix 1: functional update (doesn't need to read count)
useEffect(() => {
  const id = setInterval(() => setCount((c) => c + 1), 1000);
  return () => clearInterval(id);
}, []);

// ✅ Fix 2: include the dependency (effect re-subscribes on each change)
useEffect(() => {
  const id = setInterval(() => setCount(count + 1), 1000);
  return () => clearInterval(id);
}, [count]);

// ✅ Fix 3: ref holding the latest value (when you need to READ it in a long-lived callback)
const countRef = useRef(count);
useEffect(() => {
  countRef.current = count;
}, [count]);

useEffect(() => {
  const id = setInterval(() => console.log("latest:", countRef.current), 1000);
  return () => clearInterval(id);
}, []);
```

## 2.3 `useEffect` + closures

The effect function is a closure over the render it was created in. The **dependency array** tells React when to create a new one.

```jsx
useEffect(() => {
  document.title = `${count} clicks`;
}, [count]); // ✅ re-run with fresh closure when count changes
```

- Missing deps → stale values.
- Use `eslint-plugin-react-hooks` (`exhaustive-deps`) — don't silence it; fix the cause.
- `useEffectEvent` (React 19.2+) lets an effect call a function that always sees the latest props/state without re-running the effect.

## 2.4 Async callbacks + closures

```jsx
function Profile({ userId }) {
  const [name, setName] = useState("");

  const save = async () => {
    await api.save(userId, name);
    alert(`Saved ${name}`); // ✅ the name at click time — usually what you want
  };
}
```

A closure inside an async function keeps **click-time** values even if the user keeps typing. That's correct for "what was submitted", but wrong if you need the **current** value later → use a ref.

```jsx
// Classic stale-alert demo
function Messenger() {
  const [msg, setMsg] = useState("");
  const send = () => setTimeout(() => alert(msg), 3000); // shows msg at click time, not current
  return <><input value={msg} onChange={(e) => setMsg(e.target.value)} /><button onClick={send}>Send</button></>;
}
```

---

# 3. References & Immutability ⭐⭐⭐

## 3.1 Referential equality

React decides "did it change?" using **`Object.is`** — reference comparison for objects.

```js
Object.is(1, 1);       // true
Object.is("a", "a");   // true
Object.is({}, {});     // false — new object every time
Object.is(NaN, NaN);   // true
```

This applies to:
- `useState` / `useReducer` bail-out (same value → skip re-render)
- `useEffect` / `useMemo` / `useCallback` dependency arrays
- `React.memo` prop comparison (shallow)

## 3.2 Object / array reference problems

### ❌ Mutating state → no re-render

```jsx
const [todos, setTodos] = useState([]);

const add = (todo) => {
  todos.push(todo);  // mutation
  setTodos(todos);   // same reference → Object.is says "unchanged" → React bails out
};
```

### ❌ New references every render → effects run every render

```jsx
function Search({ query }) {
  const options = { query, limit: 10 }; // new object each render

  useEffect(() => {
    fetchResults(options);
  }, [options]); // ❗ runs after EVERY render (infinite loop if fetch sets state)
}

// ✅ depend on primitives
useEffect(() => {
  fetchResults({ query, limit: 10 });
}, [query]);

// ✅ or memoize
const options = useMemo(() => ({ query, limit: 10 }), [query]);
```

## 3.3 Immutable updates ⭐⭐⭐

```jsx
// Arrays
setTodos([...todos, newTodo]);                                    // add
setTodos(todos.filter((t) => t.id !== id));                       // remove
setTodos(todos.map((t) => (t.id === id ? { ...t, done: !t.done } : t))); // update one
setTodos([...todos].sort(byDate)); // or todos.toSorted(byDate)   // sort without mutating

// Objects
setUser({ ...user, name: "Ravi" });

// Nested
setState((prev) => ({
  ...prev,
  profile: {
    ...prev.profile,
    address: { ...prev.profile.address, city: "Delhi" },
  },
}));
```

For deeply nested state, use **Immer** (`useImmer`, Redux Toolkit) — write "mutating" code on a draft, get immutable results.

## 3.4 Shallow vs deep comparison

| Shallow (`React.memo`, default) | Deep |
| --- | --- |
| Compares each top-level prop with `Object.is` | Recursively compares all nested values |
| O(number of props) — cheap | Can be expensive on large data |
| Requires immutable updates to work | Works even with new-but-equal objects |

```js
shallowEqual({ a: 1, b: obj }, { a: 1, b: obj });      // true
shallowEqual({ a: 1, b: { x: 1 } }, { a: 1, b: { x: 1 } }); // false — different nested refs
```

Deep comparison in `React.memo(Comp, deepEqualFn)` is possible but usually a smell — fix the references instead. Implementations in `05-shallow-and-deep-copy.md`.

---

# 4. Memoization in React (summary) ⭐⭐⭐

| Tool | Memoizes | Use when |
| --- | --- | --- |
| `React.memo(Comp)` | Component render result | Expensive child re-rendering with same props |
| `useMemo(fn, deps)` | A computed value | Expensive calculation, or stable object/array reference |
| `useCallback(fn, deps)` | A function reference | Passing callbacks to memoized children or using them as deps |

```jsx
const List = React.memo(function List({ items, onSelect }) { /* expensive */ });

function Page({ data }) {
  const [filter, setFilter] = useState("");
  const [count, setCount] = useState(0);

  const items = useMemo(() => data.filter((d) => d.name.includes(filter)), [data, filter]);
  const onSelect = useCallback((id) => console.log(id), []);

  return (
    <>
      <button onClick={() => setCount((c) => c + 1)}>{count}</button>
      <List items={items} onSelect={onSelect} /> {/* skips re-render on count change */}
    </>
  );
}
```

**Trade-offs:** memoization costs memory + comparison time + complexity. Use it where profiling shows a problem; restructure first (move state down, pass `children`). Details in `10-memoization-and-caching.md`.

---

# 5. Async React ⭐⭐⭐

## 5.1 Fetching in `useEffect` — the complete pattern

```jsx
function User({ userId }) {
  const [state, setState] = useState({ status: "idle", data: null, error: null });

  useEffect(() => {
    const controller = new AbortController();
    setState({ status: "loading", data: null, error: null });

    (async () => {
      try {
        const res = await fetch(`/api/users/${userId}`, { signal: controller.signal });
        if (!res.ok) throw new Error(`HTTP ${res.status}`);
        const data = await res.json();
        setState({ status: "success", data, error: null });
      } catch (err) {
        if (err.name === "AbortError") return; // ignore cancelled requests
        setState({ status: "error", data: null, error: err });
      }
    })();

    return () => controller.abort(); // cleanup: userId changed or unmounted
  }, [userId]);

  if (state.status === "loading") return <Spinner />;
  if (state.status === "error") return <p>{state.error.message}</p>;
  return <h1>{state.data?.name}</h1>;
}
```

### Why not `useEffect(async () => ...)`?

An `async` function returns a **promise**, but `useEffect` expects the return value to be a **cleanup function** (or nothing). Define the async function inside and call it.

## 5.2 Race conditions ⭐⭐⭐

```text
userId = 1 → request A starts (slow: 2s)
userId = 2 → request B starts (fast: 0.5s)
B resolves → shows user 2 ✅
A resolves → shows user 1 ❌ (stale response overwrites the correct one)
```

### Fix 1: `ignore` flag (works for any promise)

```jsx
useEffect(() => {
  let ignore = false;

  fetchUser(userId).then((data) => {
    if (!ignore) setUser(data); // only the latest effect's result is applied
  });

  return () => {
    ignore = true; // previous effect's closure gets ignore = true
  };
}, [userId]);
```

Each effect run has **its own** `ignore` variable (closure!). Cleanup of the old run flips the old one.

### Fix 2: `AbortController` (also cancels the network request)

```jsx
useEffect(() => {
  const controller = new AbortController();
  fetch(`/api/users/${userId}`, { signal: controller.signal })
    .then((r) => r.json())
    .then(setUser)
    .catch((err) => { if (err.name !== "AbortError") setError(err); });
  return () => controller.abort();
}, [userId]);
```

| `ignore` flag | `AbortController` |
| --- | --- |
| Request still completes (wasted bandwidth) | Request actually cancelled |
| Works with any async function | Needs API support (`fetch`, axios, many SDKs) |

### Fix 3: use a data library

**TanStack Query / SWR / RTK Query** handle caching, deduping, race conditions, retries, refetch on focus, cancellation. In real apps, prefer them over hand-written fetch effects. (Frameworks also offer server components / loaders.)

## 5.3 Cleanup ⭐⭐⭐

Cleanup runs **before the next effect** and **on unmount**:

```jsx
useEffect(() => {
  const onResize = () => setWidth(window.innerWidth);
  window.addEventListener("resize", onResize);
  const id = setInterval(poll, 5000);
  const unsubscribe = socket.subscribe(onMessage);

  return () => {
    window.removeEventListener("resize", onResize);
    clearInterval(id);
    unsubscribe();
  };
}, []);
```

StrictMode (dev) mounts → unmounts → mounts to verify your cleanup works. Missing cleanup → duplicate listeners, leaks, setState on unmounted components. More in `14-memory-management.md`.

## 5.4 Async state updates & batching

- `setState` doesn't update the variable immediately — it **schedules** a re-render.

  ```jsx
  setCount(count + 1);
  console.log(count); // old value — this render's snapshot
  ```

- **Automatic batching** (React 18+): multiple state updates in the same tick — including inside promises, timeouts and native handlers — produce **one** re-render.

  ```jsx
  fetchData().then(() => {
    setLoading(false);
    setData(result); // one render, not two
  });
  ```

- After an `await`, you're in a new tick, but updates are still batched together within that tick.
- Use functional updates when multiple async completions update the same state:

  ```jsx
  results.forEach((r) => setItems((prev) => [...prev, r])); // safe accumulation
  ```

- `flushSync` forces a synchronous update (rarely needed — e.g., measuring DOM right after an update).

---

# 6. Debounce / Throttle in React (summary) ⭐⭐⭐

```jsx
// Debounce a value
function useDebounce(value, delay = 300) {
  const [debounced, setDebounced] = useState(value);
  useEffect(() => {
    const id = setTimeout(() => setDebounced(value), delay);
    return () => clearTimeout(id); // cleanup = the debounce
  }, [value, delay]);
  return debounced;
}

// Search + debounce + cancellation (no race conditions)
function Search() {
  const [query, setQuery] = useState("");
  const debouncedQuery = useDebounce(query, 300);
  const [results, setResults] = useState([]);

  useEffect(() => {
    if (!debouncedQuery.trim()) return setResults([]);
    const controller = new AbortController();
    fetch(`/api/search?q=${encodeURIComponent(debouncedQuery)}`, { signal: controller.signal })
      .then((r) => r.json())
      .then(setResults)
      .catch((e) => { if (e.name !== "AbortError") console.error(e); });
    return () => controller.abort();
  }, [debouncedQuery]);

  return (
    <>
      <input value={query} onChange={(e) => setQuery(e.target.value)} />
      <ul>{results.map((r) => <li key={r.id}>{r.title}</li>)}</ul>
    </>
  );
}
```

Pitfalls (details in `09-debounce-and-throttle.md`):
- Creating `debounce(fn)` directly in the component body → new timer every render → no debouncing.
- Memoized debounced callback with stale closure → keep the latest callback in a ref.
- Forgetting to cancel timers on unmount.
- Scroll + throttle: attach listener in `useEffect`, throttle with a stable function, `{ passive: true }`, remove on cleanup.

---

# 7. Other JS ↔ React Connections

| JS concept | React connection |
| --- | --- |
| Pure functions | Components & reducers must be pure; StrictMode double-renders to catch impurity |
| Higher-order functions | HOCs (`withAuth(Component)`), render props |
| Composition | Custom hooks composing other hooks; `children` composition |
| `this` | Class components needed binding; hooks avoid `this` |
| Event delegation | React attaches listeners at the root; synthetic events |
| `key` & identity | Keys tell React which element is which across renders; index keys break with reordering |
| Event loop | State updates batched; effects run after paint (`useEffect`) vs before paint (`useLayoutEffect`) |
| Promises / Suspense | `use(promise)` and Suspense boundaries for async data (React 19) |
| WeakMap / caching | `cache()` and memoization libraries key by reference |
| Modules | `React.lazy(() => import(...))` code splitting |

---

# 8. Output / Behaviour Questions 🧪

### Q1

```jsx
function App() {
  const [n, setN] = useState(0);
  const click = () => {
    setN(n + 1);
    setN(n + 1);
    console.log(n);
  };
  return <button onClick={click}>{n}</button>;
}
// click once
```

<details><summary>Answer</summary>

Logs `0`; button shows `1`. Both updates use the same snapshot `n = 0`; state isn't updated synchronously.

</details>

### Q2

```jsx
function App() {
  const [n, setN] = useState(0);
  const click = () => {
    setN((p) => p + 1);
    setN(n + 5);
    setN((p) => p + 1);
  };
  return <button onClick={click}>{n}</button>;
}
// click once from 0
```

<details><summary>Answer</summary>

`6` — queue: `0 → 1` (updater), then replace with `n + 5 = 5` (snapshot value), then `5 → 6` (updater).

</details>

### Q3

```jsx
function App() {
  const [items, setItems] = useState([1, 2]);
  const add = () => {
    items.push(3);
    setItems(items);
  };
  return <button onClick={add}>{items.length}</button>;
}
// click once
```

<details><summary>Answer</summary>

Still shows `2` — same array reference → React bails out of the re-render (the array is mutated in memory, though, so a later unrelated re-render would suddenly show `3`).

</details>

### Q4

```jsx
useEffect(() => {
  const id = setInterval(() => setCount(count + 1), 1000);
  return () => clearInterval(id);
}, []);
// what does the UI show after 5 seconds, starting at 0?
```

<details><summary>Answer</summary>

`1` — stale closure: every tick sets `0 + 1`.

</details>

### Q5

```jsx
const Child = React.memo(({ style }) => { console.log("child"); return null; });
function Parent() {
  const [n, setN] = useState(0);
  return <><Child style={{ color: "red" }} /><button onClick={() => setN(n + 1)} /></>;
}
// Does Child re-render on button click?
```

<details><summary>Answer</summary>

**Yes** — `style` is a new object every render, so the shallow comparison fails. Hoist it outside the component or `useMemo` it.

</details>

---

# 9. Interview Answers

**What is a stale closure in React?**

> "Each render creates new functions that close over that render's props and state. If a function created in an earlier render — like an interval callback in an effect with empty deps — keeps running, it sees outdated values. I fix it with functional state updates, correct dependency arrays, or a ref that always holds the latest value."

**How do you handle race conditions when fetching data?**

> "In the effect, I either use an `ignore` flag that the cleanup sets to true, so only the latest request's response updates state, or an `AbortController` that the cleanup aborts, which also cancels the network request. In production I'd typically use React Query or SWR, which handle this along with caching and retries."

**Why does React need immutability?**

> "React detects changes with `Object.is` reference comparison — for state bail-outs, dependency arrays and `React.memo`. If I mutate an object and set the same reference, React thinks nothing changed. Immutable updates create new references along the changed path while sharing the rest, which makes change detection cheap and reliable."

---

# 10. Key Takeaways

- Every render = a snapshot with its own props, state and functions.
- Use functional updates when new state depends on old state.
- Stale closures come from long-lived callbacks + missing deps → updater, deps, or refs.
- React compares by reference → immutable updates; avoid new objects in deps/props.
- `React.memo` + `useMemo` + `useCallback` work together; measure first.
- Fetch in effects with cleanup: `ignore` flag or `AbortController`; prefer data libraries.
- Clean up timers, listeners, subscriptions, requests.
- Debounce values with `useDebounce`; never create debounced functions in the render body.

## Readiness Checklist

- [ ] Explain "render as snapshot" with an example
- [ ] Reproduce and fix a stale closure 3 ways
- [ ] Write immutable add/remove/update/nested updates without hesitation
- [ ] Write a race-condition-safe fetch effect (both methods)
- [ ] Build search with debounce + abort
- [ ] Solve all 5 behaviour questions
