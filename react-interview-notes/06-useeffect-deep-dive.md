# 06 — useEffect Deep Dive 🔥🔥🔥

> **Interview + Understanding Guide** — Roadmap Section 6
>
> Goal: Think about effects as **synchronization**, get dependency arrays right, write correct cleanups, avoid race conditions and infinite loops — and recognise when you **don't need an effect at all**.

---

# 1. The Right Mental Model ⭐⭐⭐

❌ "useEffect is componentDidMount + componentDidUpdate + componentWillUnmount."

✅ **"useEffect synchronizes a component with an external system."**

```text
"Keep <external thing> in sync with <these props/state>."
  setup   = start synchronizing
  cleanup = stop synchronizing
  deps    = when the thing we sync with changes, stop and restart
```

```jsx
function ChatRoom({ roomId }) {
  useEffect(() => {
    const connection = createConnection(roomId);
    connection.connect();               // start syncing with roomId
    return () => connection.disconnect(); // stop syncing
  }, [roomId]);                          // roomId changes → disconnect old, connect new
}
```

Think about each effect **independently** — not about the component's lifecycle.

### What counts as an "external system"

Network, browser APIs (`document.title`, `localStorage`, `IntersectionObserver`), timers, subscriptions (WebSocket, store), third-party widgets (maps, charts), analytics.

---

# 2. Execution Timeline ⭐

```text
Mount:     render → commit DOM → paint → effect setup
Update:    render → commit DOM → paint → previous cleanup → new setup  (only if deps changed)
Unmount:   cleanup
StrictMode (dev, mount only): setup → cleanup → setup
```

- Cleanup always runs **with the values from the render it belongs to** (it's a closure).
- Effects of children run **before** effects of parents.
- Effects run in definition order within a component.

```jsx
useEffect(() => {
  console.log("setup", value);
  return () => console.log("cleanup", value);
}, [value]);
// value: 1 → 2 logs: setup 1 … cleanup 1, setup 2
```

---

# 3. Dependency Array ⭐⭐⭐

### Rule: every reactive value used inside the effect must be a dependency

**Reactive values** = props, state, and anything computed from them in the component body (variables, functions, objects).

```jsx
function Search({ query, pageSize }) {
  const [results, setResults] = useState([]);

  useEffect(() => {
    fetchResults(query, pageSize).then(setResults);
  }, [query, pageSize]); // ✅ both used → both listed
}
```

You don't **choose** dependencies — your code determines them. To change deps, change the code.

### Not dependencies

- Module-level constants and functions (outside the component)
- State setters (`setX`) and `dispatch` — stable
- `ref` objects (`ref` from `useRef` is stable; `ref.current` isn't reactive)

### Comparison is `Object.is`

```jsx
function Comp({ userId }) {
  const options = { userId, limit: 10 };   // new object every render
  useEffect(() => {
    fetchData(options);
  }, [options]); // ❌ runs after every render
}
```

**Fixes:**

```jsx
// 1. Move object creation inside the effect; depend on primitives
useEffect(() => {
  fetchData({ userId, limit: 10 });
}, [userId]);

// 2. Move static objects outside the component
const LIMIT_OPTS = { limit: 10 };

// 3. useMemo / useCallback when the object/function must be shared
const options = useMemo(() => ({ userId, limit: 10 }), [userId]);
```

### Functions as dependencies

```jsx
function Room({ roomId }) {
  function createOptions() {          // new function every render
    return { roomId, server: URL };
  }
  useEffect(() => {
    const c = connect(createOptions());
    return () => c.disconnect();
  }, [createOptions]); // ❌ reconnects every render

  // ✅ define the function INSIDE the effect, depend on roomId
}
```

### ⚠️ Don't lie about dependencies

```jsx
useEffect(() => {
  const id = setInterval(() => setCount(count + 1), 1000);
  return () => clearInterval(id);
}, []); // eslint-disable … ❌ stale closure: count is always 0
```

**Fix by removing the dependency's need:**

```jsx
useEffect(() => {
  const id = setInterval(() => setCount((c) => c + 1), 1000); // updater: no need to read count
  return () => clearInterval(id);
}, []);
```

---

# 4. Cleanup ⭐⭐⭐

Every effect that **subscribes, starts, or allocates** something must undo it.

| Setup | Cleanup |
| --- | --- |
| `setInterval` / `setTimeout` | `clearInterval` / `clearTimeout` |
| `addEventListener` | `removeEventListener` (same function reference) |
| `subscribe()` | `unsubscribe()` |
| WebSocket / connection | `close()` / `disconnect()` |
| `fetch` | `controller.abort()` or ignore flag |
| Observer (`IntersectionObserver`…) | `disconnect()` |
| Third-party widget | `widget.destroy()` |
| Animation | revert to initial state |

```jsx
useEffect(() => {
  function onResize() {
    setWidth(window.innerWidth);
  }
  window.addEventListener("resize", onResize);
  return () => window.removeEventListener("resize", onResize);
}, []);
```

### StrictMode double-invoke

In development, React mounts → unmounts → re-mounts components to verify cleanup. If your app breaks (two connections, two fetches visible, duplicate listeners), **your cleanup is missing** — don't "fix" it by using refs to skip the second run.

---

# 5. Data Fetching & Race Conditions ⭐⭐⭐

```jsx
function UserProfile({ userId }) {
  const [user, setUser] = useState(null);
  const [error, setError] = useState(null);

  useEffect(() => {
    let ignore = false;
    setUser(null);

    fetch(`/api/users/${userId}`)
      .then((r) => {
        if (!r.ok) throw new Error(r.status);
        return r.json();
      })
      .then((data) => {
        if (!ignore) setUser(data);  // ignore stale responses
      })
      .catch((err) => {
        if (!ignore) setError(err);
      });

    return () => {
      ignore = true;                  // previous request's result will be ignored
    };
  }, [userId]);
}
```

### Why the race happens

```text
userId 1 → request A (slow)
userId 2 → request B (fast) → setUser(B) ✅
request A resolves → setUser(A) ❌ wrong user displayed
```

The `ignore` flag is per-effect-run (closure), so the old run's flag flips to `true` in cleanup.

### With AbortController (also cancels the network request)

```jsx
useEffect(() => {
  const controller = new AbortController();
  (async () => {
    try {
      const res = await fetch(`/api/users/${userId}`, { signal: controller.signal });
      setUser(await res.json());
    } catch (err) {
      if (err.name !== "AbortError") setError(err);
    }
  })();
  return () => controller.abort();
}, [userId]);
```

### Why not `async` directly?

```jsx
useEffect(async () => { ... }, []); // ❌ returns a Promise; React expects a cleanup function
```

### Fetching in effects — the downsides

- No caching / deduplication; refetch on every mount
- Request waterfalls (parent fetches, then child fetches)
- No SSR support
- Boilerplate for loading/error/race handling

➡️ In real apps use **TanStack Query / SWR / RTK Query** or framework data loading (Next.js, React Router loaders). See `13-data-fetching-tanstack-query.md`.

---

# 6. You Might Not Need an Effect ⭐⭐⭐ (very common 4-YOE topic)

Effects are an **escape hatch**. If there's no external system, you probably don't need one. Unnecessary effects cause extra renders, bugs and complexity.

### 1. Transforming data for rendering → compute during render

```jsx
// ❌
const [fullName, setFullName] = useState("");
useEffect(() => setFullName(`${first} ${last}`), [first, last]);

// ✅
const fullName = `${first} ${last}`;
```

### 2. Expensive calculations → `useMemo`

```jsx
const visibleTodos = useMemo(() => getFilteredTodos(todos, filter), [todos, filter]);
```

### 3. Resetting all state when a prop changes → `key`

```jsx
// ❌ useEffect(() => setComment(""), [userId]);
<Profile userId={userId} key={userId} />
```

### 4. Adjusting some state when a prop changes → derive, or set during render

```jsx
// ❌ useEffect(() => setSelection(null), [items]);

// ✅ Better: store the selected ID and derive
const selection = items.find((i) => i.id === selectedId) ?? null;
```

### 5. Event-specific logic → event handler

```jsx
// ❌ effect fires the notification whenever product.isInCart becomes true (also on page reload!)
useEffect(() => {
  if (product.isInCart) showNotification(`Added ${product.name}`);
}, [product]);

// ✅ It's caused by a user action → put it in the handler
function handleBuyClick() {
  addToCart(product);
  showNotification(`Added ${product.name}`);
}
```

**Rule:** _Is this code running because the component was **displayed**? → effect. Because the user **did something**? → event handler._

### 6. Sending a POST request on submit → handler

```jsx
// ❌ const [jsonToSubmit, setJsonToSubmit] = useState(null);
//    useEffect(() => { if (jsonToSubmit) post(jsonToSubmit); }, [jsonToSubmit]);

// ✅
function handleSubmit(e) {
  e.preventDefault();
  post({ firstName, lastName });
}
```

(Analytics "page viewed" **is** a valid effect — caused by display.)

### 7. Chains of effects → compute in one place

```jsx
// ❌ effect sets A → effect sets B → effect sets C (multiple renders, hard to follow)
// ✅ calculate everything in the event handler or during render
```

### 8. Notifying parent about state changes → call in the same handler

```jsx
// ❌ useEffect(() => onChange(isOn), [isOn, onChange]);

// ✅
function updateToggle(next) {
  setIsOn(next);
  onChange(next);
}
// or lift state to the parent (fully controlled)
```

### 9. Passing data to the parent → let the parent fetch and pass down

### 10. Subscribing to an external store → `useSyncExternalStore`

```jsx
const isOnline = useSyncExternalStore(
  (cb) => {
    window.addEventListener("online", cb);
    window.addEventListener("offline", cb);
    return () => {
      window.removeEventListener("online", cb);
      window.removeEventListener("offline", cb);
    };
  },
  () => navigator.onLine,
  () => true, // server snapshot
);
```

### 11. Initializing the app once → module-level code or a guard

```jsx
let didInit = false;
function App() {
  useEffect(() => {
    if (!didInit) {
      didInit = true;
      loadDataFromLocalStorage();
      checkAuthToken();
    }
  }, []);
}
```

---

# 7. Infinite Loops ⭐

| Cause | Fix |
| --- | --- |
| Setting state in an effect **without deps** | Add deps |
| Setting state that's also a dependency (`[count]` + `setCount(count+1)`) | Updater, or rethink (often not an effect) |
| Object/array/function dependency recreated every render | Move inside effect, move outside component, or memoize |
| Fetch → set new array → array is a dependency | Depend on ids/primitives |

```jsx
// ❌ Infinite: new array → effect → setData(new array) → render → ...
const [data, setData] = useState([]);
const filters = { active: true };
useEffect(() => {
  fetchData(filters).then(setData);
}, [filters]);
```

---

# 8. Separating Events from Effects — `useEffectEvent`

Sometimes an effect needs to **read** the latest value of something without **re-running** when it changes.

```jsx
function ChatRoom({ roomId, theme }) {
  useEffect(() => {
    const conn = createConnection(roomId);
    conn.on("connected", () => showNotification("Connected!", theme));
    conn.connect();
    return () => conn.disconnect();
  }, [roomId, theme]); // ❌ changing theme reconnects the chat
}
```

```jsx
import { useEffect, useEffectEvent } from "react"; // stable in React 19.2

function ChatRoom({ roomId, theme }) {
  const onConnected = useEffectEvent(() => {
    showNotification("Connected!", theme); // always sees the latest theme
  });

  useEffect(() => {
    const conn = createConnection(roomId);
    conn.on("connected", () => onConnected());
    conn.connect();
    return () => conn.disconnect();
  }, [roomId]); // ✅ only reconnects when roomId changes
}
```

Pre-19.2 equivalent: the **latest ref** pattern:

```jsx
const themeRef = useRef(theme);
useLayoutEffect(() => { themeRef.current = theme; });
// read themeRef.current inside the effect callback
```

Effect Events must only be called from **inside effects**, never passed to other components or called during render.

---

# 9. Effects and Concurrent/Server Rendering

- Effects **don't run on the server**. Code depending on `window`/`localStorage` belongs in effects (or client-only components).
- Reading `localStorage` in `useState` initializer on SSR → mismatch. Pattern: render default, then update in an effect (or `useSyncExternalStore` with a server snapshot).

---

# 10. Output Questions 🧪

### Q1

```jsx
function App() {
  const [n, setN] = useState(0);
  useEffect(() => {
    console.log("effect", n);
    return () => console.log("cleanup", n);
  }, [n]);
  return <button onClick={() => setN(n + 1)}>{n}</button>;
}
```
Logs on mount (production) and after one click?

<details><summary>Answer</summary>

Mount: `effect 0`. Click: `cleanup 0`, `effect 1`. (With StrictMode in dev, mount logs `effect 0`, `cleanup 0`, `effect 0`.)

</details>

### Q2

```jsx
function Parent() {
  useEffect(() => console.log("parent effect"), []);
  return <Child />;
}
function Child() {
  useEffect(() => console.log("child effect"), []);
  return null;
}
```

<details><summary>Answer</summary>`child effect`, then `parent effect` — children's effects run first.</details>

### Q3

```jsx
function Timer() {
  const [count, setCount] = useState(0);
  useEffect(() => {
    const id = setInterval(() => setCount(count + 1), 1000);
    return () => clearInterval(id);
  }, [count]);
  return <p>{count}</p>;
}
```
Does it work? Any drawback?

<details><summary>Answer</summary>

It works (count increases), because the effect re-subscribes with the fresh `count` each time. Drawback: the interval is torn down and recreated every second. `setCount(c => c + 1)` with `[]` is cleaner.

</details>

### Q4

```jsx
function App() {
  const [data, setData] = useState([]);
  useEffect(() => {
    setData([1, 2, 3]);
  });
  return <div>{data.length}</div>;
}
```

<details><summary>Answer</summary>

**Infinite loop** — no dependency array, and each `setData([...])` creates a new array (new reference), triggering another render and another effect.

</details>

---

# 11. Interview Answers

**How do you think about useEffect?**

> "I treat it as synchronizing the component with an external system. The setup starts the synchronization, the cleanup stops it, and the dependencies — every reactive value the effect reads — decide when to stop and restart. If there's no external system involved, I usually don't need an effect: I compute derived values during render, use event handlers for user actions, and use a `key` to reset state."

**How do you prevent race conditions in data fetching?**

> "Each effect run gets its own `ignore` flag or `AbortController`. The cleanup sets the flag or aborts, so if the props change before the request finishes, the stale response is ignored. In production I'd use TanStack Query, which handles this and caching."

**Why does my effect run twice?**

> "In development with StrictMode, React mounts, unmounts and remounts components to surface missing cleanups. It doesn't happen in production. The fix is a proper cleanup, not preventing the second run."

---

# 12. Key Takeaways

- Effects synchronize with **external systems**; think per effect, not per lifecycle.
- Every reactive value used inside → dependency. Change code, not deps.
- Objects/functions in deps → move inside, move outside, or memoize.
- Always clean up subscriptions, timers, listeners, requests.
- Race conditions → ignore flag / AbortController.
- Prefer computing during render, `useMemo`, `key`, event handlers, `useSyncExternalStore` over effects.
- `useEffectEvent` to read latest values without re-triggering.

## Readiness Checklist

- [ ] Explain the sync mental model with ChatRoom
- [ ] Fix 5 "unnecessary effect" examples
- [ ] Write a race-safe fetch effect (both techniques)
- [ ] Diagnose infinite loops from object dependencies
- [ ] Explain StrictMode double effects
- [ ] Use `useEffectEvent` / latest-ref pattern
