# 08 — Custom Hooks 🔥

> **Interview + Understanding Guide** — Roadmap Section 8
>
> Goal: Design good custom hooks and implement the ~15 hooks interviewers ask for, with correct cleanup, stable references and no stale closures.

---

# 1. What is a Custom Hook?

A **custom hook** is a function whose name starts with `use` and that calls other hooks. It lets you **reuse stateful logic** (not state itself) between components.

```jsx
function useOnlineStatus() {
  const [isOnline, setIsOnline] = useState(navigator.onLine);
  useEffect(() => {
    const on = () => setIsOnline(true);
    const off = () => setIsOnline(false);
    window.addEventListener("online", on);
    window.addEventListener("offline", off);
    return () => {
      window.removeEventListener("online", on);
      window.removeEventListener("offline", off);
    };
  }, []);
  return isOnline;
}

function StatusBar() {
  const isOnline = useOnlineStatus();
  return <p>{isOnline ? "✅ Online" : "❌ Disconnected"}</p>;
}
```

### Key facts ⭐

- Each component calling a hook gets **its own independent state** — hooks share **logic**, not state.
- Must start with `use` so linters can enforce the rules of hooks.
- Re-runs on every render of the component using it (it's part of the component body).
- Functions that **don't** call hooks shouldn't be named `useX` — make them regular utilities.

### Design guidelines

- Name by **purpose**, not lifecycle: `useChatRoom`, `useDebounce` ✅ — `useMount`, `useEffectOnce` ❌ (hide intent, encourage wrong effect usage).
- Return **stable** functions (`useCallback`) and memoized objects when consumers may put them in deps.
- Return a tuple for 1–2 values (`[value, setValue]`), an object for more (`{ data, error, isLoading }`).
- Accept options objects for extensibility.
- Clean up everything you set up.
- Keep it focused — compose small hooks.

---

# 2. `useToggle`

```jsx
function useToggle(initial = false) {
  const [value, setValue] = useState(initial);
  const toggle = useCallback(() => setValue((v) => !v), []);
  return [value, toggle, setValue];
}

const [isOpen, toggleOpen] = useToggle();
```

---

# 3. `usePrevious`

```jsx
function usePrevious(value) {
  const ref = useRef();
  useEffect(() => {
    ref.current = value; // runs after render → during render ref still holds the old value
  }, [value]);
  return ref.current;
}
```

> Caveat: reading a ref during render is discouraged in strict React rules. A state-based alternative that is render-safe:

```jsx
function usePreviousState(value) {
  const [current, setCurrent] = useState(value);
  const [previous, setPrevious] = useState(undefined);
  if (value !== current) {   // adjusting state during render (allowed pattern)
    setPrevious(current);
    setCurrent(value);
  }
  return previous;
}
```

---

# 4. `useDebounce` ⭐⭐⭐

### Debounce a value

```jsx
function useDebounce(value, delay = 500) {
  const [debounced, setDebounced] = useState(value);

  useEffect(() => {
    const id = setTimeout(() => setDebounced(value), delay);
    return () => clearTimeout(id); // new value before delay → cancel previous timer
  }, [value, delay]);

  return debounced;
}

// usage
const debouncedQuery = useDebounce(query, 300);
useEffect(() => { if (debouncedQuery) search(debouncedQuery); }, [debouncedQuery]);
```

### Debounce a callback

```jsx
function useDebouncedCallback(callback, delay) {
  const callbackRef = useRef(callback);
  const timerRef = useRef(null);

  useLayoutEffect(() => {
    callbackRef.current = callback; // always call the latest callback (no stale closure)
  });

  useEffect(() => () => clearTimeout(timerRef.current), []); // cancel on unmount

  return useCallback(
    (...args) => {
      clearTimeout(timerRef.current);
      timerRef.current = setTimeout(() => callbackRef.current(...args), delay);
    },
    [delay],
  );
}
```

---

# 5. `useThrottle`

```jsx
function useThrottledCallback(callback, wait) {
  const callbackRef = useRef(callback);
  const lastRef = useRef(0);
  const timerRef = useRef(null);

  useLayoutEffect(() => {
    callbackRef.current = callback;
  });
  useEffect(() => () => clearTimeout(timerRef.current), []);

  return useCallback(
    (...args) => {
      const now = Date.now();
      const remaining = wait - (now - lastRef.current);
      if (remaining <= 0) {
        lastRef.current = now;
        callbackRef.current(...args);
      } else {
        clearTimeout(timerRef.current); // keep only the latest trailing call
        timerRef.current = setTimeout(() => {
          lastRef.current = Date.now();
          callbackRef.current(...args);
        }, remaining);
      }
    },
    [wait],
  );
}
```

---

# 6. `useFetch` ⭐⭐⭐

```jsx
function useFetch(url, options) {
  const [state, setState] = useState({ data: null, error: null, isLoading: !!url });
  const [reloadKey, setReloadKey] = useState(0);

  useEffect(() => {
    if (!url) return;
    const controller = new AbortController();
    setState((s) => ({ ...s, isLoading: true, error: null }));

    fetch(url, { ...options, signal: controller.signal })
      .then((res) => {
        if (!res.ok) throw new Error(`HTTP ${res.status}`);
        return res.json();
      })
      .then((data) => setState({ data, error: null, isLoading: false }))
      .catch((error) => {
        if (error.name === "AbortError") return; // stale or unmounted
        setState({ data: null, error, isLoading: false });
      });

    return () => controller.abort();
    // `options` deliberately not a dependency: callers often pass inline objects.
    // Pass primitives in the URL or memoize options if they must trigger refetch.
    // eslint-disable-next-line react-hooks/exhaustive-deps
  }, [url, reloadKey]);

  const refetch = useCallback(() => setReloadKey((k) => k + 1), []);
  return { ...state, refetch };
}

// usage
const { data: user, error, isLoading, refetch } = useFetch(`/api/users/${id}`);
```

Talking points: AbortController for races + unmount; status shape; `refetch`; why `options` is excluded (or require memoization); and that production apps should use **TanStack Query** (cache, dedupe, retries, background refresh).

---

# 7. `useLocalStorage` ⭐

```jsx
function useLocalStorage(key, initialValue) {
  const [value, setValue] = useState(() => {
    if (typeof window === "undefined") return initialValue; // SSR guard
    try {
      const raw = window.localStorage.getItem(key);
      return raw !== null ? JSON.parse(raw) : initialValue;
    } catch {
      return initialValue;
    }
  });

  useEffect(() => {
    try {
      window.localStorage.setItem(key, JSON.stringify(value));
    } catch {
      /* quota exceeded / private mode */
    }
  }, [key, value]);

  // Sync across tabs
  useEffect(() => {
    function onStorage(e) {
      if (e.key !== key) return;
      setValue(e.newValue !== null ? JSON.parse(e.newValue) : initialValue);
    }
    window.addEventListener("storage", onStorage);
    return () => window.removeEventListener("storage", onStorage);
    // eslint-disable-next-line react-hooks/exhaustive-deps
  }, [key]);

  return [value, setValue];
}
```

> SSR note: reading storage in the initializer causes a hydration mismatch if the server rendered `initialValue`. For SSR apps, render the default first and read storage in an effect, or use `useSyncExternalStore` with a server snapshot.

---

# 8. `useInterval` (Dan Abramov's classic) ⭐

```jsx
function useInterval(callback, delay) {
  const savedCallback = useRef(callback);

  useLayoutEffect(() => {
    savedCallback.current = callback; // latest callback, no interval reset
  });

  useEffect(() => {
    if (delay === null) return; // pause by passing null
    const id = setInterval(() => savedCallback.current(), delay);
    return () => clearInterval(id);
  }, [delay]);
}

function Counter() {
  const [count, setCount] = useState(0);
  const [running, setRunning] = useState(true);
  useInterval(() => setCount(count + 1), running ? 1000 : null); // reads fresh count every tick
  return <button onClick={() => setRunning((r) => !r)}>{count}</button>;
}
```

Why: `setInterval` + closures in effects capture stale state; this hook keeps the latest callback in a ref and only resets the interval when `delay` changes.

---

# 9. `useEventListener`

```jsx
function useEventListener(eventName, handler, target = window, options) {
  const handlerRef = useRef(handler);
  useLayoutEffect(() => {
    handlerRef.current = handler;
  });

  useEffect(() => {
    const el = target && "current" in target ? target.current : target; // ref or element
    if (!el?.addEventListener) return;
    const listener = (e) => handlerRef.current(e);
    el.addEventListener(eventName, listener, options);
    return () => el.removeEventListener(eventName, listener, options);
    // eslint-disable-next-line react-hooks/exhaustive-deps
  }, [eventName, target]);
}

useEventListener("keydown", (e) => e.key === "Escape" && close());
```

---

# 10. `useOnClickOutside` ⭐

```jsx
function useOnClickOutside(ref, handler) {
  const handlerRef = useRef(handler);
  useLayoutEffect(() => {
    handlerRef.current = handler;
  });

  useEffect(() => {
    function listener(e) {
      const el = ref.current;
      if (!el || el.contains(e.target)) return; // click inside → ignore
      handlerRef.current(e);
    }
    document.addEventListener("mousedown", listener);
    document.addEventListener("touchstart", listener);
    return () => {
      document.removeEventListener("mousedown", listener);
      document.removeEventListener("touchstart", listener);
    };
  }, [ref]);
}

function Dropdown() {
  const [open, setOpen] = useState(false);
  const ref = useRef(null);
  useOnClickOutside(ref, () => setOpen(false));
  return (
    <div ref={ref}>
      <button onClick={() => setOpen((o) => !o)}>Menu</button>
      {open && <ul role="menu">…</ul>}
    </div>
  );
}
```

Gotcha: content rendered in a **portal** is outside `ref.current` in the DOM — include the portal's ref too.

---

# 11. `useMediaQuery`

```jsx
function useMediaQuery(query) {
  const subscribe = useCallback(
    (callback) => {
      const mql = window.matchMedia(query);
      mql.addEventListener("change", callback);
      return () => mql.removeEventListener("change", callback);
    },
    [query],
  );
  return useSyncExternalStore(
    subscribe,
    () => window.matchMedia(query).matches,
    () => false, // server snapshot
  );
}

const isMobile = useMediaQuery("(max-width: 768px)");
const prefersDark = useMediaQuery("(prefers-color-scheme: dark)");
```

---

# 12. `useWindowSize`

```jsx
function useWindowSize() {
  const [size, setSize] = useState(() => ({
    width: typeof window !== "undefined" ? window.innerWidth : 0,
    height: typeof window !== "undefined" ? window.innerHeight : 0,
  }));

  useEffect(() => {
    let frame = 0;
    function onResize() {
      cancelAnimationFrame(frame);
      frame = requestAnimationFrame(() =>
        setSize({ width: window.innerWidth, height: window.innerHeight }),
      ); // rAF-throttled
    }
    window.addEventListener("resize", onResize);
    return () => {
      cancelAnimationFrame(frame);
      window.removeEventListener("resize", onResize);
    };
  }, []);

  return size;
}
```

---

# 13. `useIntersectionObserver` (lazy load / infinite scroll) ⭐

```jsx
function useIntersectionObserver({ root = null, rootMargin = "0px", threshold = 0, freezeOnceVisible = false } = {}) {
  const [entry, setEntry] = useState(null);
  const [node, setNode] = useState(null); // callback ref → re-observes when the element changes
  const frozen = freezeOnceVisible && entry?.isIntersecting;

  useEffect(() => {
    if (!node || frozen) return;
    const observer = new IntersectionObserver(([e]) => setEntry(e), { root, rootMargin, threshold });
    observer.observe(node);
    return () => observer.disconnect();
  }, [node, root, rootMargin, threshold, frozen]);

  return [setNode, entry];
}

// Infinite scroll
function Feed() {
  const { data, fetchNextPage, hasNextPage } = useInfinitePosts();
  const [sentinelRef, entry] = useIntersectionObserver({ rootMargin: "200px" });

  useEffect(() => {
    if (entry?.isIntersecting && hasNextPage) fetchNextPage();
  }, [entry, hasNextPage, fetchNextPage]);

  return (
    <>
      {data.map((p) => <Post key={p.id} post={p} />)}
      <div ref={sentinelRef} />
    </>
  );
}
```

---

# 14. `useCopyToClipboard`

```jsx
function useCopyToClipboard(resetMs = 2000) {
  const [copied, setCopied] = useState(false);
  const timer = useRef();

  useEffect(() => () => clearTimeout(timer.current), []);

  const copy = useCallback(async (text) => {
    try {
      await navigator.clipboard.writeText(text);
      setCopied(true);
      clearTimeout(timer.current);
      timer.current = setTimeout(() => setCopied(false), resetMs);
      return true;
    } catch {
      setCopied(false);
      return false;
    }
  }, [resetMs]);

  return [copied, copy];
}
```

---

# 15. `useAsync` (generic async state)

```jsx
function useAsync(asyncFn, deps = []) {
  const [state, setState] = useState({ status: "idle", data: null, error: null });

  const run = useCallback((...args) => {
    setState({ status: "pending", data: null, error: null });
    return asyncFn(...args).then(
      (data) => { setState({ status: "success", data, error: null }); return data; },
      (error) => { setState({ status: "error", data: null, error }); throw error; },
    );
    // eslint-disable-next-line react-hooks/exhaustive-deps
  }, deps);

  return { ...state, run, isLoading: state.status === "pending" };
}
```

(Add an ignore/abort guard for overlapping calls in production.)

---

# 16. The "Latest Ref" Pattern ⭐⭐⭐

Many hooks above use it. Learn it once:

```jsx
function useLatest(value) {
  const ref = useRef(value);
  useLayoutEffect(() => {
    ref.current = value;
  });
  return ref;
}
```

**Problem:** a long-lived callback (timer, listener, debounced fn) captured stale props/state.
**Solution:** keep the latest value in a ref updated after every render; the long-lived callback reads `ref.current`.
**Modern alternative:** `useEffectEvent` (React 19.2) for effects.

---

# 17. Testing Custom Hooks

```jsx
import { renderHook, act } from "@testing-library/react";

test("useToggle toggles", () => {
  const { result } = renderHook(() => useToggle());
  expect(result.current[0]).toBe(false);
  act(() => result.current[1]());
  expect(result.current[0]).toBe(true);
});

test("useDebounce", () => {
  vi.useFakeTimers();
  const { result, rerender } = renderHook(({ v }) => useDebounce(v, 300), { initialProps: { v: "a" } });
  rerender({ v: "ab" });
  expect(result.current).toBe("a");
  act(() => vi.advanceTimersByTime(300));
  expect(result.current).toBe("ab");
});
```

More in `25-testing.md`.

---

# 18. Interview Answers

**What is a custom hook and when do you write one?**

> "A function starting with `use` that composes other hooks to reuse stateful logic. Each component using it gets its own state. I extract one when the same effect/state logic appears in multiple components — subscriptions, debouncing, fetching, media queries — or to give a complex piece of component logic a clear name."

**How do you avoid stale closures in custom hooks?**

> "For long-lived callbacks like intervals, listeners or debounced functions, I keep the latest callback in a ref updated in a layout effect and have the long-lived function call `ref.current`. That way the subscription isn't recreated on every render but always sees fresh props and state. `useEffectEvent` does this natively in recent React."

---

# 19. Key Takeaways

- Custom hooks share logic, not state; name them `useX` by purpose.
- Return stable callbacks and memoized objects.
- Always clean up timers, listeners, observers, requests.
- Latest-ref pattern / `useEffectEvent` prevent stale closures.
- Use `useSyncExternalStore` for browser/external subscriptions.
- SSR-safe: guard `window`, use server snapshots.

## Practice Checklist

- [ ] useToggle · usePrevious · useDebounce (value + callback) · useThrottle
- [ ] useFetch with abort + refetch · useLocalStorage with cross-tab sync
- [ ] useInterval · useEventListener · useOnClickOutside
- [ ] useMediaQuery · useWindowSize · useIntersectionObserver
- [ ] useCopyToClipboard · useAsync · test 2 hooks with renderHook
