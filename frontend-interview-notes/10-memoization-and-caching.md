# 10 — Memoization & Caching 🔥🔥

> **Interview + Understanding Guide** — `00-starter-roadmap.md` Section 10
>
> Goal: Implement `memoize()` properly (keys, multiple args, invalidation, memory), understand recursive memoization, and know exactly when `React.memo`, `useMemo` and `useCallback` help — and when they hurt.

---

# 1. What is Memoization? ⭐⭐⭐

**Memoization** is an optimization where a function **remembers the results** of previous calls and returns the cached result when called again with the **same inputs**.

```text
memoizedFn(5)  → not in cache → compute → store {5: 120} → return 120
memoizedFn(5)  → in cache     → return 120 instantly
```

### Interview Definition

> **Memoization caches a function's return value based on its arguments, so repeated calls with the same arguments skip the computation. It trades memory for speed and is only safe for pure functions.**

### Requirements

- The function must be **pure** — same input → same output, no side effects. Memoizing `Math.random()` or a function that reads changing global state returns wrong results.
- The computation must be **expensive enough** that a cache lookup is cheaper.
- Inputs must be **repeated** often enough to get cache hits.

---

# 2. Memoization vs Caching ⭐⭐⭐

| | Memoization | Caching (general) |
| --- | --- | --- |
| Scope | A **function's** results keyed by its arguments | Any data: API responses, files, computed values |
| Where | Usually in-memory, inside a closure | Memory, `localStorage`, HTTP cache, CDN, service worker, Redis |
| Invalidation | Often none (pure functions never go stale) | Essential: TTL, versioning, events |
| Example | `memoize(fib)` | React Query caching `/api/users` for 5 minutes |

> **Memoization is a specific kind of caching.** Pure function results never become "wrong", so memoization mainly worries about **memory**; general caches also worry about **staleness**.

---

# 3. Implement `memoize()` ⭐⭐⭐

## 3.1 Single argument

```js
function memoize(fn) {
  const cache = new Map();

  return function (arg) {
    if (cache.has(arg)) return cache.get(arg); // use .has — the result may be undefined/0
    const result = fn.call(this, arg);
    cache.set(arg, result);
    return result;
  };
}

const square = memoize((n) => {
  console.log("computing", n);
  return n * n;
});

square(4); // computing 4 → 16
square(4); // 16 (cached, no log)
```

Why `Map` instead of `{}`:
- Keys can be **any type** (objects, numbers stay numbers — `{}` would turn `1` and `"1"` into the same key)
- No prototype keys like `"constructor"` / `"toString"` colliding
- `.size`, `.delete`, insertion order (useful for LRU)

## 3.2 Multiple arguments — cache keys ⭐

```js
function memoize(fn, resolver = (...args) => JSON.stringify(args)) {
  const cache = new Map();

  function memoized(...args) {
    const key = resolver(...args);
    if (cache.has(key)) return cache.get(key);
    const result = fn.apply(this, args);
    cache.set(key, result);
    return result;
  }

  memoized.cache = cache;              // expose for inspection / clearing
  memoized.clear = () => cache.clear();
  return memoized;
}

const add = memoize((a, b) => a + b);
add(1, 2); // computed
add(1, 2); // cached
```

### Cache key strategies & pitfalls

| Strategy | Problem |
| --- | --- |
| `args.join(",")` | `("a,b")` and `("a", "b")` both become `"a,b"`; `1` and `"1"` collide; objects → `"[object Object]"` |
| `JSON.stringify(args)` | Better. But: key order matters (`{a,b}` ≠ `{b,a}`), loses `undefined`/functions, slow for big objects, throws on cycles |
| Custom `resolver` | Best when you know the shape: `(user) => user.id` |
| Nested `Map`/`WeakMap` per argument (trie) | Handles object args **by reference**, no stringification |

### Object arguments by reference — `WeakMap`

```js
function memoizeByRef(fn) {
  const cache = new WeakMap(); // keys: objects; entries GC'd when the object is gone

  return function (obj) {
    if (cache.has(obj)) return cache.get(obj);
    const result = fn.call(this, obj);
    cache.set(obj, result);
    return result;
  };
}

const getStats = memoizeByRef((data) => expensiveStats(data));
```

Perfect for immutable data (like React/Redux state): same reference = same content. Libraries like **reselect** work this way.

## 3.3 Memoizing async functions

Cache the **promise**, not the value — so concurrent calls share one request (de-duplication):

```js
function memoizeAsync(fn) {
  const cache = new Map();

  return function (key) {
    if (cache.has(key)) return cache.get(key);
    const promise = fn(key).catch((err) => {
      cache.delete(key); // don't cache failures
      throw err;
    });
    cache.set(key, promise);
    return promise;
  };
}

const getUser = memoizeAsync((id) => fetch(`/api/users/${id}`).then((r) => r.json()));
getUser(1);
getUser(1); // same promise — only ONE network request
```

---

# 4. Cache Invalidation & Memory ⭐

> "There are only two hard things in Computer Science: cache invalidation and naming things." — Phil Karlton

An unbounded memo cache **grows forever** → memory leak. Strategies:

| Strategy | How |
| --- | --- |
| **Size limit (LRU)** | Evict least-recently-used when full |
| **TTL (time to live)** | Store timestamp; expire after N ms |
| **Manual clear** | `memoized.clear()` on logout, route change |
| **WeakMap keys** | Entries vanish when the key object is garbage-collected |
| **Single-entry cache** | Only remember the last call (like `useMemo`, `memoize-one`) |

## 4.1 LRU Cache ⭐ (common interview problem)

`Map` keeps **insertion order** → the first key is the least recently used if we re-insert on every access.

```js
class LRUCache {
  constructor(capacity) {
    this.capacity = capacity;
    this.map = new Map();
  }

  get(key) {
    if (!this.map.has(key)) return undefined;
    const value = this.map.get(key);
    this.map.delete(key);   // move to "most recent" end
    this.map.set(key, value);
    return value;
  }

  set(key, value) {
    if (this.map.has(key)) this.map.delete(key);
    this.map.set(key, value);
    if (this.map.size > this.capacity) {
      const oldestKey = this.map.keys().next().value; // first = least recent
      this.map.delete(oldestKey);
    }
  }
}

const lru = new LRUCache(2);
lru.set("a", 1);
lru.set("b", 2);
lru.get("a");    // touch a → order: b, a
lru.set("c", 3); // evicts b
lru.get("b");    // undefined
```

All operations O(1). (Classic DSA version: hash map + doubly linked list.)

## 4.2 Memoize with TTL

```js
function memoizeWithTTL(fn, ttl) {
  const cache = new Map();
  return function (...args) {
    const key = JSON.stringify(args);
    const hit = cache.get(key);
    if (hit && Date.now() - hit.time < ttl) return hit.value;
    const value = fn.apply(this, args);
    cache.set(key, { value, time: Date.now() });
    return value;
  };
}
```

---

# 5. Recursive Memoization ⭐

```js
// Naive Fibonacci — O(2^n): fib(40) ≈ 330 million calls
function fib(n) {
  if (n < 2) return n;
  return fib(n - 1) + fib(n - 2);
}
```

```text
             fib(5)
           /        \
       fib(4)      fib(3)   ← fib(3) computed twice, fib(2) three times...
       /    \      /    \
   fib(3) fib(2) fib(2) fib(1)
```

### ❗ Trap: wrapping doesn't memoize the recursive calls

```js
const memoFib = memoize(fib);
memoFib(40); // still slow the first time — inner calls go to the ORIGINAL `fib`
```

### ✅ Recursive calls must go through the memoized version

```js
const fastFib = memoize((n) => (n < 2 ? n : fastFib(n - 1) + fastFib(n - 2)));
fastFib(40); // instant — O(n)
```

Or with an internal cache:

```js
function fibMemo(n, memo = {}) {
  if (n < 2) return n;
  if (n in memo) return memo[n];
  return (memo[n] = fibMemo(n - 1, memo) + fibMemo(n - 2, memo));
}
```

This is **top-down dynamic programming**.

---

# 6. React Memoization ⭐⭐⭐

## 6.1 Why React needs memoization — referential equality

When a component re-renders, **everything inside it is recreated**:

```jsx
function Parent() {
  const options = { theme: "dark" };     // new object every render
  const onClick = () => console.log("hi"); // new function every render
  return <Child options={options} onClick={onClick} />;
}
```

```js
{ theme: "dark" } === { theme: "dark" } // false
(() => {}) === (() => {})               // false
```

React compares props, deps and state with **`Object.is`** (reference equality for objects). New references look like "changes".

## 6.2 `React.memo` — skip re-rendering a component

```jsx
const UserCard = React.memo(function UserCard({ user, onSelect }) {
  console.log("render", user.name);
  return <div onClick={() => onSelect(user.id)}>{user.name}</div>;
});
```

- Wraps a component; re-renders only if props **shallowly** changed (each prop compared with `Object.is`).
- Optional custom comparator: `React.memo(Comp, (prev, next) => prev.id === next.id)` (return `true` to **skip** re-render).
- Doesn't stop re-renders caused by the component's **own state or context** changes.

## 6.3 `useMemo` — memoize a computed value

```jsx
const visibleTodos = useMemo(
  () => todos.filter((t) => t.status === filter), // runs only when deps change
  [todos, filter],
);
```

Two uses:
1. **Skip expensive calculations** on re-render
2. **Keep a stable reference** for objects/arrays passed to memoized children or used in effect deps

```jsx
const style = useMemo(() => ({ color, fontSize }), [color, fontSize]);
```

## 6.4 `useCallback` — memoize a function reference

```jsx
const handleSelect = useCallback((id) => {
  setSelected(id);
}, []); // setSelected is stable

<UserCard user={u} onSelect={handleSelect} />
```

`useCallback(fn, deps)` ≡ `useMemo(() => fn, deps)`.

⚠️ `useCallback` does **not** make the function faster — it only keeps the **same reference**. It's only useful when that reference matters:
- passed to a `React.memo` child
- used in a dependency array (`useEffect`, other hooks)

## 6.5 The trio works together

```jsx
const Child = React.memo(({ onClick, config }) => { /* ... */ });

function Parent() {
  const [count, setCount] = useState(0);

  const onClick = useCallback(() => console.log("click"), []);
  const config = useMemo(() => ({ size: "lg" }), []);

  return (
    <>
      <button onClick={() => setCount((c) => c + 1)}>{count}</button>
      <Child onClick={onClick} config={config} /> {/* doesn't re-render when count changes */}
    </>
  );
}
```

Without `React.memo` on `Child`, the `useCallback`/`useMemo` here are **pointless**.

## 6.6 Dependency arrays

```jsx
useMemo(() => compute(a, b), [a, b]);
```

- Missing deps → **stale values** (stale closure bug)
- Unstable deps (objects/functions created in render) → recomputes **every render** → memoization useless
- Follow `eslint-plugin-react-hooks` (`exhaustive-deps`)

## 6.7 When memoization helps vs hurts ⭐⭐⭐

**Helps:**
- Expensive calculations (big lists, sorting/filtering thousands of items, heavy math)
- Large/expensive child components that re-render often with the same props
- Values used as effect dependencies (to avoid effect loops)
- Context provider values (`value={useMemo(() => ({ user, login }), [user])}`)

**Hurts / useless:**
- Cheap calculations — the memo bookkeeping (deps comparison, closure creation) costs more
- Props that change every render anyway
- `useCallback` passed to a non-memoized child or a DOM element (`<button onClick>`)
- Adds complexity and dependency-array bugs

> Rule of thumb: **measure first** (React DevTools Profiler). Fix structure before memoizing — move state down, lift content up (`children` prop), split components.

> **React Compiler** (React 19 era) can auto-memoize components and values at build time, reducing the need for manual `useMemo`/`useCallback`. Still know how they work — interviews ask.

---

# 7. Output / Behaviour Questions 🧪

### Q1

```js
let calls = 0;
const slowDouble = memoize((n) => { calls++; return n * 2; });
slowDouble(2); slowDouble(2); slowDouble(3); slowDouble(2);
console.log(calls);
```

<details><summary>Answer</summary>

`2` — computed for 2 and 3 only.

</details>

### Q2

```js
const getLength = memoize((arr) => arr.length, (arr) => arr); // key by reference
const a = [1, 2];
console.log(getLength(a));
a.push(3);
console.log(getLength(a));
```

<details><summary>Answer</summary>

`2` then `2` ❗ — same reference → cache hit, but the array was **mutated**. Reference-based memoization requires **immutable** data. This is exactly why React needs immutable state updates.

</details>

### Q3

```jsx
function Parent() {
  const [n, setN] = useState(0);
  const handle = useCallback(() => {}, []);
  return <><Child onX={handle} /><button onClick={() => setN(n + 1)} /></>;
}
function Child() { console.log("child render"); return null; }
```

Does `Child` re-render when the button is clicked?

<details><summary>Answer</summary>

**Yes.** `Child` isn't wrapped in `React.memo`, so it re-renders whenever its parent does, no matter how stable its props are.

</details>

---

# 8. Interview Answers

**What is memoization?**

> "Memoization caches the result of a function call keyed by its arguments, so repeated calls with the same arguments return the cached result instead of recomputing. It only works correctly for pure functions, and it trades memory for speed, so for unbounded inputs I'd add an LRU limit or TTL."

**useMemo vs useCallback vs React.memo**

> "`React.memo` wraps a component so it skips re-rendering when its props are shallowly equal. `useMemo` caches a computed value between renders until its dependencies change. `useCallback` caches a function reference. `useCallback` and `useMemo` for objects mostly matter when passing values to memoized children or using them as hook dependencies, because React compares by reference."

**When not to memoize?**

> "When the computation is cheap, when props change every render anyway, or when the child isn't memoized. Memoization has its own cost and adds dependency bugs, so I profile first and prefer restructuring components."

---

# 9. Key Takeaways

- Memoization = cache function results by arguments; pure functions only.
- Use `Map` (any key type); `has()` not truthiness; `WeakMap` for object keys.
- Multi-arg keys: `JSON.stringify` with caveats, or a custom resolver.
- Cache promises for async memoization; delete on failure.
- Unbounded caches leak → LRU / TTL / clear / WeakMap.
- Recursive memoization only works if recursive calls hit the memoized function.
- React compares by reference → `React.memo` + `useMemo` + `useCallback` work as a team; measure before using.

## Readiness Checklist

- [ ] Implement `memoize` with multi-arg keys and `has()`
- [ ] Explain cache key pitfalls
- [ ] Implement an LRU cache with `Map`
- [ ] Memoize recursive Fibonacci correctly
- [ ] Explain referential equality and why React needs memoization
- [ ] Explain when `useMemo`/`useCallback` are useless
