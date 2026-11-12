# 14 — Memory Management 🔥🔥🔥

> **Interview + Understanding Guide** — `00-starter-roadmap.md` Section 14
>
> Goal: Explain how JavaScript allocates and frees memory, recognise every common leak pattern (with fixes), and use `AbortController` and `useEffect` cleanup correctly.
>
> 📎 Companion file: **`14b-memory-leaks-devtools.md`** (your original `14b-memory-leaks-devtools.md`) covers React leak examples and the Chrome DevTools Memory tab in depth.

---

# 1. Memory Lifecycle

Every language follows the same three steps:

```text
1. Allocate   → JS does it automatically when you create values
2. Use        → read / write
3. Release    → JS does it automatically via the Garbage Collector (GC)
```

JavaScript is **garbage-collected**: you never `free()` memory. But you can still **leak** memory by keeping references to things you no longer need.

---

# 2. Stack vs Heap ⭐⭐⭐

| Stack | Heap |
| --- | --- |
| Primitives & references (pointers) in function frames | Objects, arrays, functions, closures' environments |
| Fixed size, very fast, LIFO | Large, dynamic, slower |
| Freed automatically when the function returns | Freed by the **garbage collector** when unreachable |
| Call stack overflow → `RangeError: Maximum call stack size exceeded` | Too much retained → out-of-memory crash, slow GC pauses |

```js
function createUser() {
  const id = 1;                    // stack (in the frame)
  const user = { id, tags: [] };   // reference on stack → object in heap
  return user;                     // frame popped; object survives because it's returned
}
const u = createUser();            // still reachable via `u`
```

> Mental model — engines may optimise (e.g., store small values inline), but this is the right picture for interviews.

---

# 3. Garbage Collection ⭐⭐⭐

## 3.1 Reachability

The GC keeps an object alive **only if it's reachable** from a **root**.

**Roots** include:
- Global object (`window`/`globalThis`) and global variables
- The current call stack (local variables & parameters of running functions)
- Active closures' captured variables
- Things the engine/browser holds: DOM tree, active timers, event listeners, pending promises' callbacks

```text
Roots ──▶ A ──▶ B ──▶ C        ✅ reachable → kept
          D ──▶ E              ❌ nothing points to D → collected (E too)
```

```js
let user = { name: "Asha" };
let admin = user;

user = null;  // object still reachable through admin
admin = null; // now unreachable → eligible for GC
```

## 3.2 Mark-and-Sweep (the algorithm modern engines use)

```text
1. MARK:  start from roots, follow every reference, mark everything reachable
2. SWEEP: everything NOT marked is garbage → free it
```

### Why not reference counting?

Old approach: free an object when its reference count hits 0. Fails on **cycles**:

```js
function makeCycle() {
  const a = {};
  const b = {};
  a.b = b;
  b.a = a; // each has count 1 forever under ref counting
}
makeCycle(); // with mark-and-sweep: unreachable from roots → collected ✅
```

> **Cycles are NOT a leak** in modern JS. Unreachable is unreachable, cycle or not.

## 3.3 Generational GC (V8)

Most objects die young. V8 splits the heap:
- **Young generation** — new objects; collected often and fast (Scavenger)
- **Old generation** — survivors; collected less often (Mark-Compact), incrementally/concurrently to reduce pauses

Practical impact: creating lots of short-lived objects is cheap-ish; **retaining** objects is what costs.

## 3.4 You can't force GC

There's no standard way to trigger GC from JS. You **remove references** and let the GC do its job.

---

# 4. What is a Memory Leak? ⭐⭐⭐

> **A memory leak is memory that's no longer needed by the application but is still reachable, so the garbage collector can't free it.**

Symptoms: memory grows over time (especially after repeating an action), the tab gets slower, GC pauses increase, eventually crashes.

---

# 5. Common Leak Patterns (and Fixes) ⭐⭐⭐

## 5.1 Accidental globals

```js
function track() {
  leaked = new Array(1e6); // no let/const → global (sloppy mode) → lives forever
}
```

**Fix:** use `let/const`, strict mode/modules (throws `ReferenceError`), linting.

## 5.2 Forgotten timers

```js
function startPolling(data) {
  setInterval(() => {
    console.log(data.length); // closure keeps `data` alive as long as the interval runs
  }, 1000);
}
```

Active timers are roots → their callbacks and everything they close over stay alive.

**Fix:** keep the id and clear it.

```js
const id = setInterval(poll, 1000);
// later / on unmount
clearInterval(id);
```

## 5.3 Event listeners not removed

```js
function openModal() {
  const bigState = loadHugeData();
  window.addEventListener("resize", () => layout(bigState)); // never removed
}
// Each openModal() adds another listener and retains another bigState
```

**Fix:** remove with the same reference, `{ once: true }`, or `AbortController`:

```js
const controller = new AbortController();
window.addEventListener("resize", onResize, { signal: controller.signal });
window.addEventListener("keydown", onKey, { signal: controller.signal });
// cleanup
controller.abort(); // removes both
```

## 5.4 Detached DOM elements

A DOM node removed from the page but still referenced from JS can't be collected — and it keeps its **entire subtree** alive.

```js
const cache = [];
function showRow() {
  const row = document.createElement("tr");
  table.append(row);
  cache.push(row); // reference kept
}
function clearTable() {
  table.innerHTML = ""; // rows removed from DOM, but `cache` still holds them → detached
}
```

**Fix:** drop references (`cache.length = 0`), store data instead of nodes, or use `WeakMap`/`WeakRef` for node-keyed data. DevTools heap snapshots show them as "Detached".

## 5.5 Closures retaining large data

```js
function setup() {
  const hugeArray = new Array(1e6).fill("x");
  return function getFirst() {
    return hugeArray[0]; // keeps the WHOLE array alive as long as getFirst exists
  };
}
const getFirst = setup();
```

**Fix:** capture only what you need:

```js
function setup() {
  const hugeArray = new Array(1e6).fill("x");
  const first = hugeArray[0];
  return () => first; // hugeArray can be collected
}
```

Closures aren't leaks by themselves — the leak is **keeping the closure** (e.g., in a listener/timer/global) longer than needed.

## 5.6 Unbounded caches / Maps

```js
const cache = new Map();
function getUserData(user) {
  if (!cache.has(user)) cache.set(user, compute(user)); // grows forever; keeps every user object alive
  return cache.get(user);
}
```

**Fix:** size limit (LRU), TTL, clear on logout/route change, or `WeakMap` when keyed by objects (see `10-memoization-and-caching.md`).

## 5.7 Subscriptions not cleaned up

WebSockets, `EventSource`, store subscriptions (Redux/Zustand), RxJS, observers (`IntersectionObserver`, `MutationObserver`), `BroadcastChannel`.

```js
const unsubscribe = store.subscribe(render);
// later
unsubscribe();

observer.disconnect();
socket.close();
```

## 5.8 Others

- `console.log(bigObject)` — DevTools keeps logged objects alive while the console is open.
- Growing arrays used as logs/queues that are never trimmed.
- Third-party widgets that aren't destroyed (`chart.destroy()`, `map.remove()`).

---

# 6. Weak References ⭐

## 6.1 `WeakMap` / `WeakSet`

- Keys must be **objects** (or non-registered symbols).
- Held **weakly** — if the key object becomes unreachable elsewhere, the entry is removed automatically.
- **Not iterable**, no `.size` (contents depend on GC timing).

```js
const metadata = new WeakMap();

function attachMeta(element) {
  metadata.set(element, { clicks: 0 }); // no leak: removed when element is GC'd
}
```

Use for: caching results per object, private data per instance, metadata on DOM nodes. More in `17-map-set-weakmap-weakset.md`.

## 6.2 `WeakRef` & `FinalizationRegistry` (advanced, rarely needed)

```js
const ref = new WeakRef(bigObject);
const obj = ref.deref(); // the object, or undefined if collected

const registry = new FinalizationRegistry((key) => {
  console.log(`${key} was collected`);
});
registry.register(bigObject, "bigObject");
```

GC timing is unpredictable — **don't rely on these for program logic**. Mention them as "know they exist".

---

# 7. `AbortController` ⭐⭐⭐

A standard way to **cancel** async operations.

```js
const controller = new AbortController();
const { signal } = controller;

// 1. Cancel fetch
fetch("/api/data", { signal })
  .then((r) => r.json())
  .catch((err) => {
    if (err.name === "AbortError") return; // expected — not a real error
    throw err;
  });

// 2. Remove event listeners
window.addEventListener("scroll", onScroll, { signal });

// 3. Make your own async functions abortable
function wait(ms, signal) {
  return new Promise((resolve, reject) => {
    if (signal?.aborted) return reject(signal.reason);
    const id = setTimeout(resolve, ms);
    signal?.addEventListener("abort", () => {
      clearTimeout(id);
      reject(signal.reason);
    }, { once: true });
  });
}

controller.abort(); // cancels ALL of the above
```

### Helpers

```js
fetch(url, { signal: AbortSignal.timeout(5000) });          // auto-abort after 5s
const combined = AbortSignal.any([userSignal, AbortSignal.timeout(5000)]); // either one
```

- A controller can be aborted **once**; create a new one for the next operation.
- `signal.aborted`, `signal.reason`, `abort` event.

---

# 8. `useEffect` Cleanup ⭐⭐⭐

React calls the cleanup function **before the effect re-runs** (deps changed) and **on unmount**.

```jsx
useEffect(() => {
  // setup
  return () => {
    // cleanup
  };
}, [deps]);
```

### Cleanup recipes

```jsx
// Timer
useEffect(() => {
  const id = setInterval(tick, 1000);
  return () => clearInterval(id);
}, []);

// Event listener
useEffect(() => {
  const onResize = () => setWidth(window.innerWidth);
  window.addEventListener("resize", onResize);
  return () => window.removeEventListener("resize", onResize);
}, []);

// Fetch + race condition + no setState after unmount
useEffect(() => {
  const controller = new AbortController();
  fetch(`/api/users/${id}`, { signal: controller.signal })
    .then((r) => r.json())
    .then(setUser)
    .catch((err) => {
      if (err.name !== "AbortError") setError(err);
    });
  return () => controller.abort();
}, [id]);

// Subscription
useEffect(() => {
  const unsubscribe = store.subscribe(() => setState(store.getState()));
  return unsubscribe;
}, []);

// Observer
useEffect(() => {
  const observer = new ResizeObserver(([entry]) => setSize(entry.contentRect));
  observer.observe(ref.current);
  return () => observer.disconnect();
}, []);
```

> **StrictMode** in development runs setup → cleanup → setup on mount, deliberately, so missing cleanups show up as duplicated listeners/requests.

---

# 9. Detecting Leaks with Chrome DevTools (summary)

| Tool | Use |
| --- | --- |
| **Performance monitor** | Watch JS heap size & DOM node count live |
| **Memory → Heap snapshot** | Take snapshot → repeat action → snapshot → **Comparison** view; search "Detached" |
| **Allocation instrumentation on timeline** | See what allocates over time and survives (blue bars) |
| **Allocation sampling** | Low-overhead: which functions allocate the most |
| **Retainers panel** | Why an object is still alive (the reference chain to a root) |

**Workflow:** perform the action (open/close modal) several times → force GC (trash icon) → snapshot → memory should return to baseline. If it keeps growing → inspect retainers.

- **Shallow size** — memory of the object itself
- **Retained size** — memory freed if this object were collected (includes everything only it keeps alive)

Full walkthrough: `14b-memory-leaks-devtools.md`.

> Not every memory increase is a leak — caches warming up and GC not having run yet are normal. A leak is **growth that doesn't come back down after GC across repeated actions**.

---

# 10. Interview Answers

**How does garbage collection work in JS?**

> "JavaScript uses automatic garbage collection based on reachability. Starting from roots — globals, the current call stack, active closures, the DOM — the engine marks everything reachable and sweeps the rest. That's mark-and-sweep, so circular references aren't a problem. V8 also uses generational collection, cleaning short-lived objects frequently."

**What causes memory leaks in frontend apps?**

> "Anything that keeps unneeded objects reachable: timers and intervals that aren't cleared, event listeners and subscriptions that aren't removed, detached DOM nodes still referenced in JS, closures held by long-lived callbacks capturing large data, unbounded caches, and accidental globals. In React, it's usually missing `useEffect` cleanup."

**How do you find a leak?**

> "Reproduce the suspected action repeatedly, force GC, and compare heap snapshots in the DevTools Memory tab. I look for growing object counts or detached DOM nodes and check the retainers path to see what's holding them."

---

# 11. Key Takeaways

- Stack: frames & primitives, auto-freed. Heap: objects, freed by GC.
- GC = reachability from roots; mark-and-sweep; cycles are fine.
- Leak = unneeded but still reachable.
- Usual suspects: timers, listeners, detached DOM, closures, unbounded caches, subscriptions, globals.
- `WeakMap`/`WeakSet` for object-keyed data that shouldn't prolong lifetimes.
- `AbortController` cancels fetches, removes listeners, aborts custom async work.
- `useEffect` cleanup for every subscription-like effect.

## Readiness Checklist

- [ ] Explain reachability and mark-and-sweep with a diagram
- [ ] List 6 leak sources with code and fixes
- [ ] Use AbortController for fetch, listeners, and custom promises
- [ ] Write correct cleanups for timers, listeners, fetch, observers
- [ ] Describe the heap snapshot comparison workflow
