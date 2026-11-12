# 27 — Interview Coding Patterns 🔥🔥🔥

> **Interview + Understanding Guide** — `00-starter-roadmap.md` Section 27
>
> Goal: A practice hub. Solve every problem **without looking at solutions**, using the framework below. This file adds the **browser patterns** (Event Emitter, Pub/Sub, Observer, delegation) and **mock interview rounds**; other solutions are linked.

---

# 1. How to Solve a Frontend Coding Question

```text
1. CLARIFY (1–2 min)
   - Inputs/outputs? Types? Sync or async?
   - Edge cases: empty, null, nested, duplicates, huge input, errors
   - Should it mutate? Preserve `this`? Handle cancellation?

2. EXAMPLES (1 min)
   - Write 2–3 calls with expected results — these become your tests

3. BASIC SOLUTION (5–10 min)
   - Simplest correct version; think out loud

4. EDGE CASES (3–5 min)
   - `this` + args, empty input, order, errors, cycles, holes

5. IMPROVE (if time)
   - Options (leading/trailing), cancel/flush, performance, immutability

6. COMPLEXITY
   - Time and space in Big-O

7. CONNECT
   - "In React I'd use this for…"
```

### Phrases that score points

- "Let me confirm the expected behaviour for X…"
- "I'll start with a simple version, then handle edge cases."
- "I'm using `apply` here so `this` and the arguments are preserved."
- "This is O(n) time, O(n) space because of the Set."
- "A production version would also need cancellation / a size limit / tests for…"

---

# 2. Problem Map

## Functions

| Problem | Priority | Solution |
| --- | --- | --- |
| Debounce | ⭐⭐⭐ | `09-debounce-and-throttle.md` §3 |
| Throttle | ⭐⭐⭐ | `09-debounce-and-throttle.md` §4 |
| Memoize | ⭐⭐⭐ | `10-memoization-and-caching.md` §3 |
| Once | ⭐⭐⭐ | `02b-functional-programming.md` §12 |
| Curry (+ infinite `sum(1)(2)()`) | ⭐ | `02b-functional-programming.md` §10 |
| Compose / Pipe | — | `02b-functional-programming.md` §8 |

## Objects

| Problem | Priority | Solution |
| --- | --- | --- |
| Deep clone | ⭐⭐⭐ | `05-shallow-and-deep-copy.md` §4 |
| Deep equality | ⭐⭐⭐ | `05-shallow-and-deep-copy.md` §5 |
| Flatten / Unflatten object | — | `21-utility-implementations.md` §3.4 |
| Get / Set nested property | — | `21-utility-implementations.md` §3.2–3.3 |

## Arrays

| Problem | Priority | Solution |
| --- | --- | --- |
| Flatten array | ⭐ | `04-arrays-and-data-transformation.md` §7.1 |
| GroupBy | ⭐ | `04-arrays-and-data-transformation.md` §7.3 |
| Chunk | ⭐ | `04-arrays-and-data-transformation.md` §7.7 |
| Remove / find duplicates | ⭐ | `04-arrays-and-data-transformation.md` §7.4–7.5 |
| Intersection / Difference | — | `04-arrays-and-data-transformation.md` §7.8 |

## Promises

| Problem | Priority | Solution |
| --- | --- | --- |
| Promise.all | ⭐⭐⭐ | `06-promises.md` §7.1 |
| Promise.race | ⭐ | `06-promises.md` §7.2 |
| Retry | ⭐ | `06-promises.md` §6.3 |
| Timeout | ⭐ | `06-promises.md` §6.4 |
| Promise queue | ⭐ | `06-promises.md` §6.6 |
| Concurrency limiter | ⭐ | `06-promises.md` §6.5 |

## Polyfills

| Problem | Priority | Solution |
| --- | --- | --- |
| map / filter / reduce / forEach | ⭐⭐⭐ | `22-polyfills.md` §2 |
| call / apply / bind | ⭐⭐⭐ | `22-polyfills.md` §3 |

## Browser

| Problem | Priority | Solution |
| --- | --- | --- |
| Event emitter | ⭐ | **This file** §3 |
| Event delegation | — | **This file** §4 (+ `12-dom-and-browser-apis.md` §4.5) |
| Pub/Sub | — | **This file** §5 |
| Observer pattern | — | **This file** §6 |

---

# 3. Event Emitter ⭐

Implement `on`, `off`, `once`, `emit` (like Node's `EventEmitter`).

```js
class EventEmitter {
  #events = new Map(); // eventName → Set of listeners

  on(event, listener) {
    if (typeof listener !== "function") throw new TypeError("Listener must be a function");
    if (!this.#events.has(event)) this.#events.set(event, new Set());
    this.#events.get(event).add(listener);
    return () => this.off(event, listener); // return an unsubscribe function
  }

  off(event, listener) {
    const listeners = this.#events.get(event);
    if (!listeners) return this;
    listeners.delete(listener);
    if (listeners.size === 0) this.#events.delete(event); // avoid empty entries (memory)
    return this;
  }

  once(event, listener) {
    const wrapper = (...args) => {
      this.off(event, wrapper);   // remove BEFORE calling (handles re-entrant emits)
      listener.apply(this, args);
    };
    wrapper.original = listener;  // so off(event, listener) can find it
    return this.on(event, wrapper);
  }

  emit(event, ...args) {
    const listeners = this.#events.get(event);
    if (!listeners || listeners.size === 0) return false;
    // Copy so listeners added/removed during emit don't affect this round
    [...listeners].forEach((listener) => listener.apply(this, args));
    return true;
  }

  removeAllListeners(event) {
    event === undefined ? this.#events.clear() : this.#events.delete(event);
    return this;
  }

  listenerCount(event) {
    return this.#events.get(event)?.size ?? 0;
  }
}
```

Support `off(event, originalFn)` for `once` listeners:

```js
// inside off(), before listeners.delete(listener):
for (const l of listeners) {
  if (l.original === listener) listeners.delete(l);
}
```

### Tests

```js
const emitter = new EventEmitter();
const log = (msg) => console.log("got", msg);

const unsubscribe = emitter.on("message", log);
emitter.emit("message", "hello");   // got hello
unsubscribe();
emitter.emit("message", "again");   // (nothing)

emitter.once("ready", () => console.log("ready!"));
emitter.emit("ready"); // ready!
emitter.emit("ready"); // (nothing)
```

### Discussion points

- **Set vs Array:** Set prevents duplicate registrations of the same function and gives O(1) delete. Array allows duplicates (Node's behaviour) — mention the choice.
- **Copy before iterating** so `once` handlers removing themselves don't skip the next listener.
- **Error isolation:** one throwing listener stops the rest; wrap each in `try/catch` if needed.
- **Return an unsubscribe function** — perfect for `useEffect` cleanup.
- **Memory leaks:** listeners that aren't removed keep their closures alive.

---

# 4. Event Delegation (coding version)

> "Implement a function `delegate(parent, eventType, selector, handler)`."

```js
function delegate(parent, eventType, selector, handler) {
  function listener(event) {
    const target = event.target.closest(selector);
    if (target && parent.contains(target)) {
      handler.call(target, event, target); // `this` = matched element, like jQuery
    }
  }
  parent.addEventListener(eventType, listener);
  return () => parent.removeEventListener(eventType, listener); // cleanup
}

// Usage: one listener handles every current AND future .delete-btn
const stop = delegate(document.querySelector("#todo-list"), "click", ".delete-btn", (e, btn) => {
  btn.closest("li").remove();
});
```

Key points: `closest()` for nested elements (icons inside buttons), `contains()` so matches outside the parent are ignored, return a remover. Theory in `12-dom-and-browser-apis.md` §4.5.

---

# 5. Pub/Sub

**Publish–Subscribe**: publishers and subscribers don't know each other; they communicate through a **broker** using topics.

```js
const PubSub = (() => {
  const topics = new Map();
  let nextId = 0;

  return {
    subscribe(topic, callback) {
      if (!topics.has(topic)) topics.set(topic, new Map());
      const id = nextId++;
      topics.get(topic).set(id, callback);
      return { unsubscribe: () => topics.get(topic)?.delete(id) };
    },

    publish(topic, data) {
      const subscribers = topics.get(topic);
      if (!subscribers) return 0;
      [...subscribers.values()].forEach((cb) => {
        try {
          cb(data);
        } catch (err) {
          console.error(`Subscriber error on "${topic}":`, err); // isolate failures
        }
      });
      return subscribers.size;
    },
  };
})();

// Cart widget
const sub = PubSub.subscribe("cart:updated", ({ count }) => (badge.textContent = count));
// Product page (doesn't know the badge exists)
PubSub.publish("cart:updated", { count: 3 });
sub.unsubscribe();
```

### Event Emitter vs Pub/Sub vs Observer

| Pattern | Coupling | Shape |
| --- | --- | --- |
| **Observer** | Subject **knows** its observers directly | `subject.subscribe(observer)` → `observer.update(data)` |
| **Event Emitter** | Listeners attach to a specific emitter object, by event name | `emitter.on("x", fn)` |
| **Pub/Sub** | Fully decoupled via a broker/channel; neither side knows the other | `bus.publish("topic", data)` |

Real-world: DOM events (observer/emitter), Redux store subscriptions (observer), message brokers/`BroadcastChannel`/micro-frontend event buses (pub/sub), RxJS (observables).

---

# 6. Observer Pattern

A **subject** maintains a list of **observers** and notifies them when its state changes.

```js
class Observable {
  #observers = new Set();
  #state;

  constructor(initialState) {
    this.#state = initialState;
  }

  subscribe(observer) {
    this.#observers.add(observer);
    observer(this.#state); // optional: emit current state immediately
    return () => this.#observers.delete(observer);
  }

  setState(updater) {
    const next = typeof updater === "function" ? updater(this.#state) : updater;
    if (Object.is(next, this.#state)) return; // no change → no notification
    this.#state = next;
    this.#observers.forEach((observer) => observer(this.#state));
  }

  getState() {
    return this.#state;
  }
}

// A tiny global store
const themeStore = new Observable("light");
const unsubscribe = themeStore.subscribe((theme) => (document.body.dataset.theme = theme));
themeStore.setState("dark");
```

### Connect it to React (how Zustand/Redux work under the hood)

```jsx
function useStore(store) {
  return useSyncExternalStore(
    (onChange) => store.subscribe(onChange), // subscribe → returns unsubscribe
    () => store.getState(),                  // snapshot
  );
}

function ThemeToggle() {
  const theme = useStore(themeStore);
  return <button onClick={() => themeStore.setState((t) => (t === "light" ? "dark" : "light"))}>{theme}</button>;
}
```

(Note: `subscribe` above calls the observer immediately; `useSyncExternalStore` tolerates that because it re-reads the snapshot.)

---

# 7. More Practice Problems (frequently asked)

Try these on your own; hints given.

| # | Problem | Hint |
| --- | --- | --- |
| 1 | **LRU Cache** with `get`/`put` O(1) | `Map` insertion order — `10-memoization-and-caching.md` §4.1 |
| 2 | **`sleep(ms)`** and **`delay` every item** in a loop | `new Promise(r => setTimeout(r, ms))` + `for...of` + `await` |
| 3 | **Implement `setInterval` with `setTimeout`** | Recursive timeout; return a cancel function |
| 4 | **Promisify** a callback API | `22-polyfills.md`, `06-promises.md` §7.5 |
| 5 | **Async task runner in series** `runSeries([t1, t2])` | `reduce` on `Promise.resolve()` or `for...of` + `await` |
| 6 | **Auto-retry fetch with timeout** | Combine retry + `AbortSignal.timeout` |
| 7 | **Typeahead / autocomplete** component | Debounce + AbortController + keyboard nav + highlighting |
| 8 | **Infinite scroll** | IntersectionObserver on a sentinel |
| 9 | **Star rating** / **accordion** / **tabs** / **modal** | State + accessibility (aria, focus trap for modal, Esc to close) |
| 10 | **Progress bar** that animates and queues | rAF or CSS transitions + queue |
| 11 | **Count-down timer** with pause/resume | Store end time; `setInterval` + cleanup; avoid drift |
| 12 | **Virtualized list** | `23-javascript-performance.md` §10 |
| 13 | **Implement `classNames`** | `21-utility-implementations.md` §5 |
| 14 | **Template string interpolation** `render("Hi {{name}}", data)` | Regex replace — `20-regular-expressions.md` §7 |
| 15 | **Flatten DOM tree to text / find all text nodes** | Recursion over `childNodes` or `TreeWalker` |
| 16 | **`getElementsByClassName` polyfill** | DFS over `children`, `classList.contains` |
| 17 | **Detect cycle in object / safe `JSON.stringify`** | `WeakSet` of visited objects |
| 18 | **Implement `useDebounce`, `usePrevious`, `useLocalStorage`, `useFetch`** | `09`, `13`, `26` files |
| 19 | **Undo/redo stack** | Two stacks (past/future) + immutable snapshots |
| 20 | **Throttled API calls with max N requests per second** | Token bucket / queue + timestamps |

### Two quick reference solutions

```js
// usePrevious
function usePrevious(value) {
  const ref = useRef();
  useEffect(() => {
    ref.current = value;
  }, [value]);
  return ref.current; // value from the previous render
}

// Safe JSON.stringify (replace circular refs)
function safeStringify(obj) {
  const seen = new WeakSet();
  return JSON.stringify(obj, (key, value) => {
    if (value && typeof value === "object") {
      if (seen.has(value)) return "[Circular]";
      seen.add(value);
    }
    return value;
  });
}
```

---

# 8. Mock Interview Rounds ⏱️

Simulate real conditions: timer on, no notes, explain out loud, then self-review against the solution files.

### Round 1 — JS Fundamentals (45 min)

1. Output question: event loop with `async/await` + `setTimeout` (`08-event-loop.md` Q3)
2. Implement `debounce` with `this` support
3. Implement `Array.prototype.myReduce`
4. Explain closures + a stale closure example in React

### Round 2 — Async (45 min)

1. Implement `Promise.all`
2. Implement `promisePool(tasks, limit)`
3. Implement `retry(fn, retries, delay)` with backoff
4. Explain race conditions in `useEffect` and fix one

### Round 3 — Objects & `this` (45 min)

1. Implement `deepClone` with circular references
2. Implement `myBind` (with `new` support)
3. Implement `myNew` and `myInstanceOf`
4. Output questions on `this` (`11-this-and-function-binding.md` Q3, Q4, Q8)

### Round 4 — Browser / Machine Coding (60–90 min)

1. Build an autocomplete with debounce, request cancellation, keyboard navigation, and loading/error states
2. Use event delegation for the result list
3. Discuss performance (virtualization for 10k results) and accessibility (`role="listbox"`, `aria-activedescendant`)

### Round 5 — Patterns (45 min)

1. Implement `EventEmitter` with `on/off/once/emit`
2. Implement an `LRUCache`
3. Implement `curry` + infinite `sum`
4. Design question: "How would you share state between two unrelated components without a library?" (Observer + `useSyncExternalStore`, or Context)

---

# 9. Self-Review Rubric

After each problem, score yourself:

| Area | ✅ |
| --- | --- |
| Clarified requirements & edge cases first | |
| Correct basic solution | |
| Handled `this`, args, empty input, errors | |
| Clean naming and structure | |
| Stated time/space complexity | |
| Mentioned production improvements | |
| Connected to a real React use case | |
| Explained clearly while coding | |

---

# 10. Final Checklist (from `00-starter-roadmap.md`)

- [ ] Functions: debounce, throttle, memoize, once, curry, compose, pipe
- [ ] Objects: deep clone, deep equality, flatten, unflatten, get, set
- [ ] Arrays: flatten, groupBy, chunk, unique, duplicates, intersection, difference
- [ ] Promises: all, race, retry, timeout, queue, concurrency limiter
- [ ] Polyfills: map, filter, reduce, forEach, call, apply, bind
- [ ] Browser: event emitter, event delegation, pub/sub, observer
- [ ] Completed all 5 mock rounds under time pressure
