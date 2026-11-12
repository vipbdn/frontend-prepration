# 21 — JavaScript Utility Implementations 🔥🔥🔥

> **Interview + Understanding Guide** — `00-starter-roadmap.md` Section 21
>
> Goal: One place to revise **every utility** interviewers ask you to implement. Utilities explained in depth elsewhere are given here in compact form with a pointer; the rest (nested get/set, unflatten, shuffle, pick/omit, etc.) are fully explained here.
>
> **How to practise:** cover the code, read the signature + test cases, implement from scratch, then compare.

---

# Index

| Utility | Priority | Full explanation |
| --- | --- | --- |
| `debounce`, `throttle` | ⭐⭐⭐ | `09-debounce-and-throttle.md` |
| `memoize` | ⭐⭐⭐ | `10-memoization-and-caching.md` |
| `once`, `curry`, `compose`, `pipe`, `partial` | ⭐⭐⭐ / ⭐ | `02b-functional-programming.md` |
| `Promise.all / race / any / allSettled` | ⭐⭐⭐ | `06-promises.md` §7 |
| Retry, timeout, queue, concurrency limiter | ⭐ | `06-promises.md` §6 |
| `deepClone`, `deepEqual` | ⭐⭐⭐ | `05-shallow-and-deep-copy.md` |
| `flatten`, `groupBy`, `chunk`, `unique`, `difference`, `intersection`, flatten object | ⭐ | `04-arrays-and-data-transformation.md` |
| `getNestedValue`, `setNestedValue`, unflatten object, `shuffle`, `pick/omit`, `isEmpty`, `classNames`, `sleep` | ⭐ | **This file** |

---

# 1. Function Utilities (compact)

```js
// debounce — run after `delay` ms of silence
function debounce(fn, delay) {
  let timer;
  return function (...args) {
    clearTimeout(timer);
    timer = setTimeout(() => fn.apply(this, args), delay);
  };
}

// throttle — run at most once per `limit` ms (leading)
function throttle(fn, limit) {
  let last = 0;
  return function (...args) {
    const now = Date.now();
    if (now - last >= limit) {
      last = now;
      fn.apply(this, args);
    }
  };
}

// memoize — cache by arguments
function memoize(fn, resolver = (...args) => JSON.stringify(args)) {
  const cache = new Map();
  return function (...args) {
    const key = resolver(...args);
    if (!cache.has(key)) cache.set(key, fn.apply(this, args));
    return cache.get(key);
  };
}

// once — run only the first time
function once(fn) {
  let called = false, result;
  return function (...args) {
    if (!called) { called = true; result = fn.apply(this, args); }
    return result;
  };
}

// curry — f(a, b, c) → f(a)(b)(c) / f(a, b)(c) / ...
function curry(fn, arity = fn.length) {
  return function curried(...args) {
    if (args.length >= arity) return fn.apply(this, args);
    return (...next) => curried.apply(this, [...args, ...next]);
  };
}

// compose (right → left) & pipe (left → right)
const compose = (...fns) => (x) => fns.reduceRight((acc, fn) => fn(acc), x);
const pipe = (...fns) => (x) => fns.reduce((acc, fn) => fn(acc), x);

// partial — pre-fill arguments
const partial = (fn, ...preset) => (...later) => fn(...preset, ...later);
```

**Test cases**

```js
const add3 = curry((a, b, c) => a + b + c);
console.assert(add3(1)(2)(3) === 6 && add3(1, 2)(3) === 6);
console.assert(pipe((x) => x + 1, (x) => x * 2)(5) === 12);
console.assert(compose((x) => x + 1, (x) => x * 2)(5) === 11);
let n = 0; const inc = once(() => ++n); inc(); inc();
console.assert(n === 1);
```

---

# 2. Promise Utilities (compact)

```js
const sleep = (ms) => new Promise((r) => setTimeout(r, ms));

function promiseAll(iterable) {
  return new Promise((resolve, reject) => {
    const items = [...iterable];
    const results = new Array(items.length);
    let remaining = items.length;
    if (!remaining) return resolve(results);
    items.forEach((p, i) =>
      Promise.resolve(p).then((v) => {
        results[i] = v;
        if (--remaining === 0) resolve(results);
      }, reject),
    );
  });
}

function promiseRace(iterable) {
  return new Promise((resolve, reject) => {
    for (const p of iterable) Promise.resolve(p).then(resolve, reject);
  });
}

function promiseAllSettled(iterable) {
  return promiseAll(
    [...iterable].map((p) =>
      Promise.resolve(p).then(
        (value) => ({ status: "fulfilled", value }),
        (reason) => ({ status: "rejected", reason }),
      ),
    ),
  );
}

function promiseAny(iterable) {
  return new Promise((resolve, reject) => {
    const items = [...iterable];
    const errors = new Array(items.length);
    let remaining = items.length;
    if (!remaining) return reject(new AggregateError([], "All promises were rejected"));
    items.forEach((p, i) =>
      Promise.resolve(p).then(resolve, (e) => {
        errors[i] = e;
        if (--remaining === 0) reject(new AggregateError(errors, "All promises were rejected"));
      }),
    );
  });
}

async function retry(fn, retries = 3, delay = 300) {
  for (let attempt = 0; ; attempt++) {
    try {
      return await fn(attempt);
    } catch (err) {
      if (attempt >= retries) throw err;
      await sleep(delay * 2 ** attempt); // exponential backoff
    }
  }
}

function withTimeout(promise, ms) {
  let id;
  const timeout = new Promise((_, reject) => {
    id = setTimeout(() => reject(new Error(`Timeout after ${ms}ms`)), ms);
  });
  return Promise.race([promise, timeout]).finally(() => clearTimeout(id));
}

async function promisePool(tasks, limit) {
  const results = new Array(tasks.length);
  let next = 0;
  const worker = async () => {
    while (next < tasks.length) {
      const i = next++;
      results[i] = await tasks[i]();
    }
  };
  await Promise.all(Array.from({ length: Math.min(limit, tasks.length) }, worker));
  return results;
}
```

---

# 3. Object Utilities

## 3.1 `deepClone` & `deepEqual` (compact)

```js
function deepClone(value, seen = new WeakMap()) {
  if (value === null || typeof value !== "object") return value;
  if (seen.has(value)) return seen.get(value);
  if (value instanceof Date) return new Date(value);
  if (value instanceof RegExp) return new RegExp(value.source, value.flags);
  if (value instanceof Map) {
    const m = new Map();
    seen.set(value, m);
    value.forEach((v, k) => m.set(deepClone(k, seen), deepClone(v, seen)));
    return m;
  }
  if (value instanceof Set) {
    const s = new Set();
    seen.set(value, s);
    value.forEach((v) => s.add(deepClone(v, seen)));
    return s;
  }
  const copy = Array.isArray(value) ? [] : Object.create(Object.getPrototypeOf(value));
  seen.set(value, copy);
  for (const key of Reflect.ownKeys(value)) copy[key] = deepClone(value[key], seen);
  return copy;
}

function deepEqual(a, b) {
  if (Object.is(a, b)) return true;
  if (typeof a !== "object" || typeof b !== "object" || !a || !b) return false;
  if (Object.getPrototypeOf(a) !== Object.getPrototypeOf(b)) return false;
  if (a instanceof Date) return a.getTime() === b.getTime();
  const keysA = Reflect.ownKeys(a);
  const keysB = Reflect.ownKeys(b);
  if (keysA.length !== keysB.length) return false;
  return keysA.every((k) => Object.hasOwn(b, k) && deepEqual(a[k], b[k]));
}
```

(Full versions with Map/Set equality and cycle handling in `05-shallow-and-deep-copy.md`.)

## 3.2 `getNestedValue` (like `_.get`) ⭐

```js
/**
 * get(obj, "a.b[0].c", defaultValue)
 */
function get(obj, path, defaultValue) {
  const keys = Array.isArray(path)
    ? path
    : path.replace(/\[(\w+)\]/g, ".$1").split(".").filter(Boolean); // "a[0].b" → ["a","0","b"]

  let result = obj;
  for (const key of keys) {
    if (result === null || result === undefined) return defaultValue;
    result = result[key];
  }
  return result === undefined ? defaultValue : result;
}

const data = { user: { addresses: [{ city: "Pune" }] } };
get(data, "user.addresses[0].city");      // "Pune"
get(data, ["user", "addresses", 0, "city"]); // "Pune"
get(data, "user.phone.number", "N/A");    // "N/A"
get(data, "user.addresses.length");       // 1
```

> Modern code often uses optional chaining: `data?.user?.addresses?.[0]?.city ?? "N/A"`. The utility is useful when the **path is dynamic** (a string from config, form field names, table column definitions).

## 3.3 `setNestedValue` (like `_.set`) ⭐

### Mutable version

```js
function set(obj, path, value) {
  const keys = Array.isArray(path)
    ? path
    : path.replace(/\[(\w+)\]/g, ".$1").split(".").filter(Boolean);

  let current = obj;
  keys.forEach((key, i) => {
    if (i === keys.length - 1) {
      current[key] = value;
      return;
    }
    if (current[key] === null || typeof current[key] !== "object") {
      // create array if the NEXT key is numeric, else object
      current[key] = /^\d+$/.test(keys[i + 1]) ? [] : {};
    }
    current = current[key];
  });
  return obj;
}

const o = {};
set(o, "a.b[0].c", 1);
// { a: { b: [ { c: 1 } ] } }
```

### Immutable version (React-friendly) ⭐

```js
function setIn(obj, path, value) {
  const keys = Array.isArray(path)
    ? path
    : path.replace(/\[(\w+)\]/g, ".$1").split(".").filter(Boolean);

  if (keys.length === 0) return value;
  const [head, ...rest] = keys;
  const base = obj ?? (/^\d+$/.test(head) ? [] : {});
  const copy = Array.isArray(base) ? [...base] : { ...base };
  copy[head] = setIn(base[head], rest, value);
  return copy;
}

const state = { form: { user: { name: "Asha", city: "Pune" } }, other: {} };
const next = setIn(state, "form.user.city", "Delhi");

next.form.user.city;          // "Delhi"
state.form.user.city;         // "Pune" — original untouched
next.other === state.other;   // true — structural sharing
```

## 3.4 Flatten & Unflatten object ⭐

```js
function flattenObject(obj, prefix = "", out = {}) {
  for (const [key, value] of Object.entries(obj)) {
    const path = prefix ? `${prefix}.${key}` : key;
    if (value && typeof value === "object" && !Array.isArray(value) && Object.keys(value).length) {
      flattenObject(value, path, out);
    } else {
      out[path] = value;
    }
  }
  return out;
}

function unflattenObject(flat) {
  const result = {};
  for (const [path, value] of Object.entries(flat)) {
    const keys = path.split(".");
    let current = result;
    keys.forEach((key, i) => {
      if (i === keys.length - 1) current[key] = value;
      else current = current[key] ??= {};
    });
  }
  return result;
}

const nested = { a: 1, b: { c: 2, d: { e: 3 } } };
const flat = flattenObject(nested);   // { a: 1, "b.c": 2, "b.d.e": 3 }
unflattenObject(flat);                // { a: 1, b: { c: 2, d: { e: 3 } } }
```

Real-world: form libraries (field names like `"address.city"`), i18n keys, query params, MongoDB-style updates.

## 3.5 `pick` / `omit`

```js
const pick = (obj, keys) =>
  Object.fromEntries(keys.filter((k) => k in obj).map((k) => [k, obj[k]]));

const omit = (obj, keys) => {
  const exclude = new Set(keys);
  return Object.fromEntries(Object.entries(obj).filter(([k]) => !exclude.has(k)));
};

const user = { id: 1, name: "Asha", password: "secret" };
pick(user, ["id", "name"]); // { id: 1, name: "Asha" }
omit(user, ["password"]);   // { id: 1, name: "Asha" }

// Inline alternative for omit
const { password, ...safeUser } = user;
```

## 3.6 `isEmpty`

```js
function isEmpty(value) {
  if (value == null) return true;                                   // null / undefined
  if (typeof value === "string" || Array.isArray(value)) return value.length === 0;
  if (value instanceof Map || value instanceof Set) return value.size === 0;
  if (typeof value === "object") return Object.keys(value).length === 0;
  return false; // numbers, booleans, functions
}
```

---

# 4. Array Utilities

```js
// flatten with depth
const flatten = (arr, depth = Infinity) =>
  arr.reduce(
    (acc, x) =>
      Array.isArray(x) && depth > 0 ? acc.concat(flatten(x, depth - 1)) : acc.concat([x]),
    [],
  );

// groupBy (key or function)
const groupBy = (arr, keyOrFn) => {
  const getKey = typeof keyOrFn === "function" ? keyOrFn : (x) => x[keyOrFn];
  return arr.reduce((acc, x) => {
    (acc[getKey(x)] ||= []).push(x);
    return acc;
  }, {});
};

// chunk
const chunk = (arr, size) =>
  Array.from({ length: Math.ceil(arr.length / size) }, (_, i) =>
    arr.slice(i * size, i * size + size),
  );

// unique / uniqueBy
const unique = (arr) => [...new Set(arr)];
const uniqueBy = (arr, key) => [...new Map(arr.map((x) => [x[key], x])).values()]; // keeps LAST

// difference / intersection (O(n + m))
const difference = (a, b) => {
  const s = new Set(b);
  return a.filter((x) => !s.has(x));
};
const intersection = (a, b) => {
  const s = new Set(b);
  return [...new Set(a.filter((x) => s.has(x)))];
};
```

## 4.1 `shuffle` — Fisher–Yates ⭐

```js
function shuffle(array) {
  const arr = [...array]; // don't mutate input
  for (let i = arr.length - 1; i > 0; i--) {
    const j = Math.floor(Math.random() * (i + 1)); // 0 ≤ j ≤ i
    [arr[i], arr[j]] = [arr[j], arr[i]];
  }
  return arr;
}
```

❌ The popular `arr.sort(() => Math.random() - 0.5)` is **biased** (not all permutations equally likely) and O(n log n). Fisher–Yates is unbiased and O(n). Interviewers love this follow-up.

## 4.2 `range`, `zip`, `countBy`

```js
const range = (start, end, step = 1) =>
  Array.from({ length: Math.max(Math.ceil((end - start) / step), 0) }, (_, i) => start + i * step);
range(0, 5);     // [0, 1, 2, 3, 4]
range(0, 10, 3); // [0, 3, 6, 9]

const zip = (...arrays) =>
  Array.from({ length: Math.min(...arrays.map((a) => a.length)) }, (_, i) => arrays.map((a) => a[i]));
zip([1, 2], ["a", "b"]); // [[1, "a"], [2, "b"]]

const countBy = (arr, fn) =>
  arr.reduce((acc, x) => {
    const k = fn(x);
    acc[k] = (acc[k] || 0) + 1;
    return acc;
  }, {});
countBy([1.1, 1.9, 2.5], Math.floor); // { 1: 2, 2: 1 }
```

---

# 5. Frontend-Specific Utilities

```js
// classNames / clsx
function classNames(...args) {
  return args
    .flatMap((arg) => {
      if (!arg) return [];
      if (typeof arg === "string" || typeof arg === "number") return [arg];
      if (Array.isArray(arg)) return [classNames(...arg)];
      return Object.entries(arg).filter(([, on]) => on).map(([cls]) => cls);
    })
    .filter(Boolean)
    .join(" ");
}
classNames("btn", { active: true, disabled: false }, ["lg", null]); // "btn active lg"

// Query string ↔ object
const toQuery = (obj) => new URLSearchParams(obj).toString();
const fromQuery = (qs) => Object.fromEntries(new URLSearchParams(qs));
toQuery({ q: "react hooks", page: 2 }); // "q=react+hooks&page=2"

// Format currency / numbers (use Intl, not regex)
new Intl.NumberFormat("en-IN", { style: "currency", currency: "INR" }).format(1234567.5);
// "₹12,34,567.50"

// Relative time
new Intl.RelativeTimeFormat("en", { numeric: "auto" }).format(-1, "day"); // "yesterday"

// Truncate
const truncate = (s, n) => (s.length > n ? s.slice(0, n - 1) + "…" : s);
```

---

# 6. Interview Strategy for "Implement X" Questions

```text
1. Clarify    — inputs, outputs, edge cases (empty, null, nested, huge, async?)
2. Signature  — write it + 2–3 test calls first
3. Basic      — simplest correct version; talk while coding
4. Edge cases — this binding, args, empty input, errors, order, cycles
5. Improve    — options (leading/trailing), cancel, performance, immutability
6. Complexity — state time/space (Big-O)
7. Real use   — where you'd use it in a React app
```

---

# 7. Key Takeaways

- Most utilities = **closures** (state between calls) + **HOFs** (wrap a function) + **recursion** (nested data).
- Always preserve `this` and arguments in function wrappers (`fn.apply(this, args)`).
- Promise utilities: wrap with `Promise.resolve`, store by index, count completions, handle empty input.
- Nested object utilities: parse the path, walk/create levels, copy each level for immutable versions.
- Use `Set`/`Map` for O(1) lookups; Fisher–Yates for unbiased shuffle.

## Practice Tracker (from `00-starter-roadmap.md`)

- [ ] Function: debounce, throttle, memoize, once, curry, compose, pipe, partial
- [ ] Promise: all, race, any, allSettled, retry, timeout, queue, concurrency limiter
- [ ] Object: deepClone, deepEqual, get, set (mutable + immutable), flatten, unflatten
- [ ] Array: flatten, groupBy, chunk, unique, difference, intersection, shuffle
