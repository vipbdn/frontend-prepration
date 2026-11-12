# 09 — Debounce & Throttle 🔥🔥🔥

> **Interview + Understanding Guide** — `00-starter-roadmap.md` Section 9
>
> Goal: Explain the difference instantly, implement both from scratch (basic → leading/trailing/cancel), and use them correctly in React without stale closures or leaked timers.
>
> **Prerequisites:** closures (`02a-closures.md`), higher-order functions (`02b-functional-programming.md`), event loop timers (`08-event-loop.md`).

---

# 1. The Problem

Some events fire **extremely often**:

| Event | Frequency |
| --- | --- |
| `input` / `keyup` | every keystroke |
| `scroll` | dozens of times per second |
| `resize` | continuously while dragging |
| `mousemove` / `pointermove` | 60+ times per second |

Running expensive work (API calls, layout calculations, re-renders) on every event wastes network, CPU and battery, and can make the UI janky.

**Debounce** and **throttle** are higher-order functions that **limit how often a function runs**.

---

# 2. Debounce vs Throttle ⭐⭐⭐

```text
Events:     x x x x x x x       x x x       x
            ───────────────────────────────────────▶ time

Debounce:                 ●             ●           ●
(trailing)   waits for a quiet period, runs ONCE after the burst

Throttle:   ●     ●     ●       ●     ●     ●
             runs at most once every N ms DURING the burst
```

| | Debounce | Throttle |
| --- | --- | --- |
| Idea | "Wait until they **stop**" | "At most once **every N ms**" |
| Runs during continuous activity? | ❌ No (trailing) — keeps postponing | ✅ Yes, at a steady rate |
| Typical use | Search/autocomplete, form validation, autosave, resize end | Scroll position, infinite scroll, mousemove/drag, resize while dragging, rate-limiting clicks |
| Analogy | Elevator doors: each new person resets the close timer | A machine gun with a fixed fire rate |

> **One-liner:** _Debounce groups a burst into one call at the end. Throttle spreads calls out evenly during the burst._

---

# 3. Debounce

## 3.1 Basic implementation ⭐⭐⭐

```js
function debounce(fn, delay) {
  let timerId;

  return function (...args) {
    clearTimeout(timerId);                 // cancel the previous scheduled call
    timerId = setTimeout(() => {
      fn.apply(this, args);                // run with latest args and correct `this`
    }, delay);
  };
}
```

```js
const search = debounce((q) => console.log("API call:", q), 300);

search("r");
search("re");
search("rea");
search("react");
// ...300ms of silence...
// API call: react   ← only once, with the latest value
```

### How it works

```text
call("r")      → set timer T1 (300ms)
call("re")     → clear T1, set T2
call("rea")    → clear T2, set T3
call("react")  → clear T3, set T4
(300ms quiet)  → T4 fires → fn("react")
```

### Why each piece matters (interviewers ask)

| Piece | Why |
| --- | --- |
| `timerId` in closure | Persists between calls; shared by all calls of this debounced function |
| `clearTimeout` | Cancels the pending call so only the last survives |
| `...args` | Uses the **latest** arguments |
| Regular `function` (not arrow) for the wrapper | So `this` is whatever the debounced function is called on |
| Arrow inside `setTimeout` | Captures the wrapper's `this` and `args` |
| `fn.apply(this, args)` | Preserves `this` + arguments |

```js
const obj = {
  name: "Asha",
  log: debounce(function () {
    console.log(this.name);
  }, 100),
};
obj.log(); // "Asha" ✅ — `this` preserved
```

## 3.2 Leading vs Trailing

- **Trailing** (default): run **after** the burst ends.
- **Leading**: run **immediately on the first call**, then ignore calls until the burst ends (quiet for `delay` ms).

```text
Events:    x x x x x            x x
Leading:   ●                    ●
Trailing:          ●                  ●
Both:      ●       ●            ●     ●   (trailing only if more calls came after the leading one)
```

Leading is good for **buttons** (respond instantly, ignore double-clicks).

## 3.3 Full implementation (leading, trailing, cancel, flush) ⭐

```js
function debounce(fn, delay, { leading = false, trailing = true } = {}) {
  let timerId = null;
  let lastArgs;
  let lastThis;

  function invoke() {
    const args = lastArgs;
    const ctx = lastThis;
    lastArgs = lastThis = undefined; // consume — prevents a duplicate trailing call
    return fn.apply(ctx, args);
  }

  function debounced(...args) {
    lastArgs = args;
    lastThis = this;

    const isNewBurst = timerId === null;
    clearTimeout(timerId);

    timerId = setTimeout(() => {
      timerId = null;
      if (trailing && lastArgs) invoke(); // only if calls happened after the leading call
      lastArgs = lastThis = undefined;
    }, delay);

    if (leading && isNewBurst) invoke();
  }

  debounced.cancel = () => {
    clearTimeout(timerId);
    timerId = null;
    lastArgs = lastThis = undefined;
  };

  debounced.flush = () => {
    if (timerId === null) return;
    clearTimeout(timerId);
    timerId = null;
    if (lastArgs) invoke(); // run the pending call right now
  };

  return debounced;
}
```

- **`cancel()`** — drop the pending call (e.g., component unmount, user cleared the input).
- **`flush()`** — run the pending call immediately (e.g., save draft before navigating away).

## 3.4 Debounce returning a promise (follow-up)

```js
function debounceAsync(fn, delay) {
  let timerId;
  let pending = [];

  return (...args) =>
    new Promise((resolve, reject) => {
      clearTimeout(timerId);
      pending.push({ resolve, reject });
      timerId = setTimeout(async () => {
        const callers = pending;
        pending = [];
        try {
          const result = await fn(...args);
          callers.forEach((c) => c.resolve(result));
        } catch (err) {
          callers.forEach((c) => c.reject(err));
        }
      }, delay);
    });
}
```

All callers in a burst receive the result of the single final call.

## 3.5 Use cases

- **Search API / autocomplete** — call only after typing pauses
- **Form validation** — validate "username available" after typing stops
- **Autosave** drafts
- **Window resize** — recalculate layout once resizing ends

---

# 4. Throttle

## 4.1 Basic implementation — timestamp (leading) ⭐⭐⭐

```js
function throttle(fn, limit) {
  let lastCall = 0;

  return function (...args) {
    const now = Date.now();
    if (now - lastCall >= limit) {
      lastCall = now;
      fn.apply(this, args);
    }
  };
}
```

Runs immediately, then ignores calls until `limit` ms pass. ⚠️ The **last** event in a burst may be dropped — bad if you need the final scroll position.

## 4.2 Timer-flag version

```js
function throttle(fn, limit) {
  let waiting = false;

  return function (...args) {
    if (waiting) return;
    fn.apply(this, args);
    waiting = true;
    setTimeout(() => {
      waiting = false;
    }, limit);
  };
}
```

## 4.3 Leading + trailing (guarantees the final call) ⭐

```js
function throttle(fn, wait) {
  let lastCall = 0;
  let timerId = null;
  let lastArgs;
  let lastThis;

  return function throttled(...args) {
    const now = Date.now();
    const remaining = wait - (now - lastCall);
    lastArgs = args;
    lastThis = this;

    if (remaining <= 0) {
      // Enough time has passed → run now (leading edge)
      clearTimeout(timerId);
      timerId = null;
      lastCall = now;
      fn.apply(this, args);
    } else if (!timerId) {
      // Too soon → schedule one trailing call with the latest args
      timerId = setTimeout(() => {
        lastCall = Date.now();
        timerId = null;
        fn.apply(lastThis, lastArgs);
      }, remaining);
    }
  };
}
```

```text
wait = 100ms
t=0    call → run (leading)
t=30   call → schedule trailing at t=100
t=60   call → already scheduled; update lastArgs
t=100  trailing runs with args from t=60
```

## 4.4 Throttle with `requestAnimationFrame`

For visual updates (scroll-linked animations, drag), sync with the screen's refresh rate instead of a fixed ms:

```js
function rafThrottle(fn) {
  let scheduled = false;
  let lastArgs;

  return function (...args) {
    lastArgs = args;
    if (scheduled) return;
    scheduled = true;
    requestAnimationFrame(() => {
      scheduled = false;
      fn.apply(this, lastArgs);
    });
  };
}

window.addEventListener("scroll", rafThrottle(updateProgressBar));
```

## 4.5 Use cases

- **Scroll** — scroll position, sticky headers, infinite scroll checks (or use `IntersectionObserver` instead)
- **Resize** while dragging
- **Mouse movement** / drag & drop
- **Game loops / shooting** rate limits
- **Analytics** — send at most one event per N seconds

> For "is this element visible?" checks, `IntersectionObserver` is better than throttled scroll handlers — no main-thread work on every scroll.

---

# 5. Debounce & Throttle in React ⭐⭐⭐

## 5.1 ❌ The common bug — recreated on every render

```jsx
function Search() {
  const [query, setQuery] = useState("");

  // ❌ New debounced function (new timer!) every render → doesn't debounce at all
  const debouncedSearch = debounce((q) => fetchResults(q), 300);

  const onChange = (e) => {
    setQuery(e.target.value);  // re-render → new debounce
    debouncedSearch(e.target.value);
  };
  return <input value={query} onChange={onChange} />;
}
```

Every render creates a new closure with its own `timerId`, so previous timers are never cleared.

## 5.2 ✅ Option A — debounce the **value** with `useDebounce` (most common)

```jsx
function useDebounce(value, delay = 500) {
  const [debounced, setDebounced] = useState(value);

  useEffect(() => {
    const id = setTimeout(() => setDebounced(value), delay);
    return () => clearTimeout(id); // cleanup cancels the previous timer on every change + unmount
  }, [value, delay]);

  return debounced;
}
```

```jsx
function Search() {
  const [query, setQuery] = useState("");
  const debouncedQuery = useDebounce(query, 300);

  useEffect(() => {
    if (!debouncedQuery) return;
    const controller = new AbortController();

    fetch(`/api/search?q=${encodeURIComponent(debouncedQuery)}`, { signal: controller.signal })
      .then((r) => r.json())
      .then(setResults)
      .catch((err) => {
        if (err.name !== "AbortError") setError(err);
      });

    return () => controller.abort(); // cancel stale request → no race condition
  }, [debouncedQuery]);

  return <input value={query} onChange={(e) => setQuery(e.target.value)} />;
}
```

The input stays responsive (state updates on every keystroke); only the **effect** is debounced.

**How cleanup does the debouncing:** each keystroke changes `value` → React runs the previous effect's cleanup (`clearTimeout`) → schedules a new timer. Only the last timer survives.

## 5.3 ✅ Option B — debounce the **callback** (`useDebouncedCallback`)

```jsx
function useDebouncedCallback(callback, delay) {
  const callbackRef = useRef(callback);

  // Always point to the latest callback → avoids stale closures
  useLayoutEffect(() => {
    callbackRef.current = callback;
  });

  // Create the debounced function ONCE (per delay)
  const debounced = useMemo(
    () => debounce((...args) => callbackRef.current(...args), delay),
    [delay],
  );

  // Cancel pending call on unmount
  useEffect(() => () => debounced.cancel(), [debounced]);

  return debounced;
}
```

```jsx
const save = useDebouncedCallback((text) => api.saveDraft(id, text), 1000);
<textarea onChange={(e) => save(e.target.value)} />;
```

### Why each piece

| Piece | Problem it solves |
| --- | --- |
| `useMemo` (or `useRef`) | Same debounced instance across renders → timer persists |
| `callbackRef` | The memoized debounce would otherwise call a **stale** callback that captured old props/state |
| `cancel` on unmount | Prevents calling `setState` / API after unmount (leak) |

## 5.4 Stale closure trap

```jsx
const [count, setCount] = useState(0);

// ❌ Debounced once with useCallback([]) → always sees count = 0
const logCount = useCallback(debounce(() => console.log(count), 500), []);
```

Fixes: the ref pattern above, functional updates (`setCount(c => c + 1)`), or include deps (which recreates the debounce — then you need cleanup).

## 5.5 Throttle in React

```jsx
function useThrottledCallback(callback, wait) {
  const callbackRef = useRef(callback);
  useLayoutEffect(() => {
    callbackRef.current = callback;
  });
  return useMemo(() => throttle((...args) => callbackRef.current(...args), wait), [wait]);
}

function ScrollTracker() {
  const [y, setY] = useState(0);
  const onScroll = useThrottledCallback(() => setY(window.scrollY), 100);

  useEffect(() => {
    window.addEventListener("scroll", onScroll, { passive: true });
    return () => window.removeEventListener("scroll", onScroll); // same reference → removed
  }, [onScroll]);

  return <div>Scrolled: {y}px</div>;
}
```

> React 18+ also offers `useDeferredValue` — it doesn't reduce API calls, but it keeps typing responsive by rendering expensive results at lower priority. Debounce = fewer calls; `useDeferredValue` = smoother rendering.

---

# 6. Output / Behaviour Questions 🧪

### Q1

```js
const d = debounce((x) => console.log(x), 100);
d(1);
d(2);
setTimeout(() => d(3), 50);
setTimeout(() => d(4), 200);
```

<details><summary>Answer</summary>

Logs `3` (at ~150ms) then `4` (at ~300ms). The call at 50ms resets the timer; 100ms of quiet follows; the call at 200ms starts a new burst.

</details>

### Q2

```js
const t = throttle((x) => console.log(x), 100); // leading-only timestamp version
t(1);
t(2);
t(3);
setTimeout(() => t(4), 150);
```

<details><summary>Answer</summary>

`1` immediately, `4` at ~150ms. Calls 2 and 3 are dropped (no trailing in the basic version).

</details>

### Q3

```js
const obj = {
  name: "Asha",
  greet: debounce(() => console.log(this?.name), 50),
};
obj.greet();
```

<details><summary>Answer</summary>

Not `"Asha"`: you get `undefined` in an ES module (or `""` in a classic browser script, where `this` is `window` and `window.name` is an empty string). The function passed to `debounce` is an **arrow**, so `this` comes from where it was written, not from `obj`. Use a regular `function`.

</details>

---

# 7. Interview Answers

**Debounce vs throttle**

> "Both limit how often a function runs. Debounce waits until events stop for a given delay and then runs once — good for search inputs and validation. Throttle runs at most once per interval while events keep firing — good for scroll, resize and mouse movement."

**How does your debounce work?**

> "It returns a wrapper that keeps a timer id in a closure. Each call clears the existing timer and starts a new one with the latest arguments, so only the last call in a burst actually runs. I use `apply` to preserve `this` and the arguments."

**Debounce in React — what goes wrong?**

> "Creating the debounced function in the component body creates a new timer every render, so it never debounces. I either debounce the value with a `useDebounce` hook that sets a timeout in `useEffect` and clears it in cleanup, or memoize the debounced callback with `useMemo`/`useRef`, keep the latest callback in a ref to avoid stale closures, and cancel it on unmount."

---

# 8. Key Takeaways

- Debounce = run after quiet period; throttle = run at most every N ms.
- Both are HOFs built on closures and timers.
- Preserve `this` and `args` with `apply`.
- Leading/trailing options; `cancel`/`flush` for lifecycle control.
- `rafThrottle` for visual updates; `IntersectionObserver` for visibility.
- React: never create debounced functions directly in render; use `useDebounce` or memoized callback + ref; clean up timers; abort stale requests.

## Readiness Checklist

- [ ] Draw the debounce vs throttle timeline
- [ ] Implement basic debounce and throttle in < 3 minutes each
- [ ] Implement debounce with leading/trailing/cancel
- [ ] Implement throttle with trailing call
- [ ] Write `useDebounce` and `useDebouncedCallback` and explain each line
- [ ] Explain the stale closure + recreated-debounce bugs in React
