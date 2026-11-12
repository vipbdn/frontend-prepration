# 05 — Shallow Copy & Deep Copy 🔥

> **Interview + Understanding Guide** — `00-starter-roadmap.md` Section 5
>
> Goal: Understand references and mutation precisely, know every copying technique and its limits, and implement `deepClone` (with circular references) and `deepEqual` from scratch.

---

# 1. Primitive vs Reference (Recap)

| Primitive (copied by value) | Reference (copied by reference) |
| --- | --- |
| `string`, `number`, `bigint`, `boolean`, `undefined`, `null`, `symbol` | `object`, `array`, `function`, `Date`, `Map`, `Set`… |
| Immutable | Mutable |
| Compared by value | Compared by reference |

```js
let a = 10;
let b = a;
b = 20;
console.log(a); // 10 — independent copy

const x = { v: 1 };
const y = x;
y.v = 2;
console.log(x.v); // 2 — same object
```

### Stack vs Heap (mental model)

```text
Stack (fixed-size, fast)        Heap (dynamic)
a = 10
b = 20
x ──────────────┐
y ──────────────┴────────▶  { v: 2 }
```

> Engines optimise freely, so "primitives on stack, objects on heap" is a **mental model**, not a spec rule. It's the right level for interviews.

### Mutation vs Immutability

```js
// Mutation — changes existing object (everyone holding the reference sees it)
user.name = "Ravi";
arr.push(4);

// Immutable update — new object, old untouched
const nextUser = { ...user, name: "Ravi" };
const nextArr = [...arr, 4];
```

**Why it matters in React:** React decides whether to re-render by comparing references (`Object.is`). Mutating state keeps the same reference → React thinks nothing changed → **no re-render**.

```js
// ❌ Bug: same reference
items.push(newItem);
setItems(items);

// ✅ New reference
setItems([...items, newItem]);
```

---

# 2. Shallow Copy ⭐⭐⭐

A **shallow copy** creates a new top-level object, but **nested objects are still shared**.

```js
const original = { name: "Asha", address: { city: "Pune" } };
const copy = { ...original };

copy.name = "Ravi";           // ✅ only copy changes
copy.address.city = "Delhi";  // ❗ changes original too

console.log(original.name);         // "Asha"
console.log(original.address.city); // "Delhi"
console.log(original.address === copy.address); // true
```

```text
original ──▶ { name: "Asha", address: ─┐ }
                                        ├──▶ { city: "Delhi" }
copy     ──▶ { name: "Ravi", address: ─┘ }
```

### Shallow copy techniques

| Objects | Arrays |
| --- | --- |
| `{ ...obj }` | `[...arr]` |
| `Object.assign({}, obj)` | `arr.slice()` |
| | `Array.from(arr)` |
| | `arr.concat()` |

### Spread operator limitations ⭐⭐⭐

- **Shallow** — nested objects shared
- Copies only **own enumerable** properties
- **Loses the prototype** — class instances become plain objects
- Getters are **evaluated** — you get the value, not the getter
- Non-enumerable props (and property descriptors) are lost

```js
class User {
  constructor(n) { this.name = n; }
  greet() { return "hi"; }
}
const u = { ...new User("Asha") };
u.greet; // undefined ❗ prototype lost
```

### `Object.assign` limitations ⭐⭐⭐

Everything spread has, plus:
- **Mutates** the first argument
- Invokes **setters** on the target

### Updating nested state immutably

Copy **every level on the path** you change:

```js
const state = {
  user: { profile: { name: "Asha", city: "Pune" }, roles: ["dev"] },
  theme: "dark",
};

const next = {
  ...state,
  user: {
    ...state.user,
    profile: { ...state.user.profile, city: "Delhi" },
  },
};

next.user.roles === state.user.roles; // true — untouched branches are shared (structural sharing ✅)
```

> This is called **structural sharing** — it's efficient and it's exactly what React wants. Libraries like **Immer** (used by Redux Toolkit) let you "mutate" a draft and produce this result automatically.

---

# 3. Deep Copy ⭐⭐⭐

A **deep copy** recursively copies **every** level → no shared references.

## 3.1 `JSON.parse(JSON.stringify(obj))`

```js
const deep = JSON.parse(JSON.stringify(original));
```

Works for plain data, but has many limitations:

| Input | After JSON round-trip |
| --- | --- |
| `Date` | ISO **string** ❗ |
| `undefined` (in object) | **property removed** |
| `undefined` (in array) | `null` |
| Functions | removed |
| `Symbol` keys/values | removed |
| `NaN`, `Infinity` | `null` |
| `Map`, `Set` | `{}` |
| `RegExp` | `{}` |
| `BigInt` | **TypeError** |
| Circular reference | **TypeError** |
| Class instances | plain objects (prototype lost) |

```js
const data = { d: new Date(0), u: undefined, f() {}, n: NaN, m: new Map([[1, 2]]) };
JSON.parse(JSON.stringify(data));
// { d: "1970-01-01T00:00:00.000Z", n: null, m: {} }
```

## 3.2 `structuredClone()` ⭐⭐⭐ (modern answer)

Built into browsers and Node 17+.

```js
const original = {
  date: new Date(),
  map: new Map([["a", 1]]),
  set: new Set([1, 2]),
  nested: { arr: [1, 2, { x: 1 }] },
};
original.self = original; // circular

const clone = structuredClone(original);
clone.date instanceof Date; // true ✅
clone.map.get("a");         // 1 ✅
clone.self === clone;       // true ✅ circular refs preserved
clone.nested !== original.nested; // true ✅
```

**Supports:** primitives, plain objects, arrays, `Date`, `RegExp`, `Map`, `Set`, `ArrayBuffer`/typed arrays, `Blob`, `File`, `Error` types, circular references.

**Limitations:**
- ❌ **Functions** → throws `DataCloneError`
- ❌ **DOM nodes** → throws
- ❌ **Symbols** → throws
- ❌ **Prototype / class identity lost** → instances become plain objects
- ❌ Getters/setters and property descriptors not preserved

> Same algorithm used by `postMessage` (Web Workers) and IndexedDB — see `24-web-workers.md`.

## 3.3 Comparison

| | Spread / assign | JSON | `structuredClone` | Custom `deepClone` | `_.cloneDeep` |
| --- | --- | --- | --- | --- | --- |
| Depth | Shallow | Deep | Deep | Deep | Deep |
| Date | shared ref | string ❌ | ✅ | ✅ (if handled) | ✅ |
| Map / Set | shared ref | `{}` ❌ | ✅ | ✅ (if handled) | ✅ |
| Functions | shared ref | dropped | throws | shared ref | shared ref |
| Circular | n/a | throws | ✅ | ✅ (with WeakMap) | ✅ |
| `undefined` | ✅ | dropped | ✅ | ✅ | ✅ |

---

# 4. Implement `deepClone()` ⭐⭐⭐

### Step 1 — Basic (objects + arrays)

```js
function deepClone(value) {
  // Primitives and functions are returned as-is
  if (value === null || typeof value !== "object") return value;

  const copy = Array.isArray(value) ? [] : {};
  for (const key of Object.keys(value)) {
    copy[key] = deepClone(value[key]);
  }
  return copy;
}
```

### Step 2 — Handle circular references

Without it, `obj.self = obj` → infinite recursion → `RangeError: Maximum call stack size exceeded`.

**Idea:** remember objects we've already cloned in a `WeakMap` (original → clone). If we see one again, return the existing clone.

### Step 3 — Final interview version

```js
function deepClone(value, seen = new WeakMap()) {
  // 1. Primitives & functions
  if (value === null || typeof value !== "object") return value;

  // 2. Circular reference check
  if (seen.has(value)) return seen.get(value);

  // 3. Special built-ins
  if (value instanceof Date) return new Date(value.getTime());
  if (value instanceof RegExp) return new RegExp(value.source, value.flags);

  if (value instanceof Map) {
    const map = new Map();
    seen.set(value, map);
    value.forEach((v, k) => map.set(deepClone(k, seen), deepClone(v, seen)));
    return map;
  }

  if (value instanceof Set) {
    const set = new Set();
    seen.set(value, set);
    value.forEach((v) => set.add(deepClone(v, seen)));
    return set;
  }

  // 4. Arrays & objects — keep the prototype
  const copy = Array.isArray(value) ? [] : Object.create(Object.getPrototypeOf(value));
  seen.set(value, copy); // ⚠️ register BEFORE recursing, or cycles still loop

  // 5. Own keys including symbols
  for (const key of Reflect.ownKeys(value)) {
    copy[key] = deepClone(value[key], seen);
  }

  return copy;
}
```

### Testing it

```js
const original = {
  num: 1,
  str: "a",
  undef: undefined,
  date: new Date(0),
  regex: /ab+c/gi,
  arr: [1, { deep: true }],
  map: new Map([["k", { v: 1 }]]),
  set: new Set([1, 2]),
  [Symbol("id")]: 123,
  fn() { return "hi"; },
};
original.self = original;

const clone = deepClone(original);

clone.self === clone;                  // true — cycle preserved
clone.arr[1] !== original.arr[1];      // true — deep
clone.date.getTime() === 0;            // true
clone.map.get("k") !== original.map.get("k"); // true
clone.fn === original.fn;              // true — functions shared (by design)
"undef" in clone;                      // true
```

### Talking points interviewers want

- **Why `WeakMap` and not `Map`?** Keys are held weakly → the tracking map doesn't keep originals alive after cloning; garbage-collection friendly. Also a `WeakMap` only accepts objects, which is exactly what we track.
- **Why set `seen` before recursing?** So a child pointing back to the parent finds the in-progress clone.
- **Functions** are usually shared — cloning closures isn't meaningful.
- **`Reflect.ownKeys`** includes symbol and non-enumerable keys; `Object.keys` would miss symbols.
- **Recursion depth:** extremely deep structures could overflow the stack; an iterative version with an explicit stack fixes that.

---

# 5. Implement `deepEqual()` ⭐

```js
function deepEqual(a, b, seen = new WeakMap()) {
  // Same reference or same primitive (Object.is handles NaN and ±0)
  if (Object.is(a, b)) return true;

  // If either is not an object, and they weren't equal above → not equal
  if (typeof a !== "object" || typeof b !== "object" || a === null || b === null) {
    return false;
  }

  // Different prototypes → different types ([] vs {}, Date vs object)
  if (Object.getPrototypeOf(a) !== Object.getPrototypeOf(b)) return false;

  // Circular reference guard
  if (seen.get(a) === b) return true;
  seen.set(a, b);

  if (a instanceof Date) return a.getTime() === b.getTime();
  if (a instanceof RegExp) return a.source === b.source && a.flags === b.flags;

  if (a instanceof Map || a instanceof Set) {
    if (a.size !== b.size) return false;
    if (a instanceof Set) {
      for (const v of a) if (!b.has(v)) return false; // primitive members
      return true;
    }
    for (const [k, v] of a) {
      if (!b.has(k) || !deepEqual(v, b.get(k), seen)) return false;
    }
    return true;
  }

  const keysA = Reflect.ownKeys(a);
  const keysB = Reflect.ownKeys(b);
  if (keysA.length !== keysB.length) return false;

  for (const key of keysA) {
    if (!Object.hasOwn(b, key) || !deepEqual(a[key], b[key], seen)) return false;
  }
  return true;
}

deepEqual({ a: [1, { b: 2 }] }, { a: [1, { b: 2 }] }); // true
deepEqual({ a: 1 }, { a: 1, b: undefined });           // false — different keys
deepEqual(NaN, NaN);                                   // true
deepEqual([1, 2], { 0: 1, 1: 2 });                     // false — different prototypes
```

### Shallow equal (what `React.memo` does)

```js
function shallowEqual(a, b) {
  if (Object.is(a, b)) return true;
  if (typeof a !== "object" || typeof b !== "object" || !a || !b) return false;
  const keysA = Object.keys(a);
  const keysB = Object.keys(b);
  if (keysA.length !== keysB.length) return false;
  return keysA.every((k) => Object.hasOwn(b, k) && Object.is(a[k], b[k]));
}
```

---

# 6. When to Use What

| Situation | Use |
| --- | --- |
| React state update | Spread at each changed level (or Immer) |
| Plain JSON data from an API | `structuredClone` (or JSON if you know it's plain) |
| Data with Date/Map/Set/cycles | `structuredClone` |
| Need functions / class instances preserved | Custom `deepClone` or `_.cloneDeep` |
| Interview | Explain trade-offs, then implement `deepClone` |

> **Avoid deep cloning by default.** It's expensive and in React it creates new references everywhere, which defeats `React.memo`. Prefer targeted immutable updates.

---

# 7. Output Questions 🧪

### Q1

```js
const a = { x: 1, y: { z: 1 } };
const b = { ...a };
b.x = 2;
b.y.z = 2;
console.log(a.x, a.y.z);
```

<details><summary>Answer</summary>

`1 2` — top-level copied, nested shared.

</details>

### Q2

```js
const obj = { d: new Date(0), u: undefined, n: null, f: () => 1 };
console.log(JSON.parse(JSON.stringify(obj)));
```

<details><summary>Answer</summary>

`{ d: '1970-01-01T00:00:00.000Z', n: null }` — date becomes string; `undefined` and function dropped; `null` kept.

</details>

### Q3

```js
console.log(JSON.stringify([undefined, () => {}, NaN]));
```

<details><summary>Answer</summary>

`[null,null,null]` — inside arrays, unsupported values become `null` (to keep indexes).

</details>

### Q4

```js
const arr = [[1], [2]];
const copy = [...arr];
copy[0].push(99);
copy[1] = [3];
console.log(arr);
```

<details><summary>Answer</summary>

`[[1, 99], [2]]` — mutating a shared inner array affects both; reassigning an index of the copy does not.

</details>

### Q5

```js
try {
  structuredClone({ fn() {} });
} catch (e) {
  console.log(e.name);
}
```

<details><summary>Answer</summary>

`DataCloneError` — functions can't be structured-cloned.

</details>

### Q6

```js
const o = { a: 1 };
o.self = o;
JSON.stringify(o);
```

<details><summary>Answer</summary>

`TypeError: Converting circular structure to JSON`

</details>

---

# 8. Interview Answers

**Shallow vs deep copy**

> "A shallow copy creates a new top-level object but nested objects are still shared references, so changing a nested value affects both. A deep copy recursively copies every level so nothing is shared. Spread and `Object.assign` are shallow; `structuredClone` is the modern built-in deep copy."

**Why not always use JSON for deep copy?**

> "It only works for plain JSON data. Dates become strings, `undefined` and functions are dropped, `Map`/`Set` become empty objects, `NaN` becomes `null`, and circular references or BigInt throw. `structuredClone` handles most of those except functions, DOM nodes and class prototypes."

**How does your deepClone handle cycles?**

> "I keep a `WeakMap` from original objects to their clones. Before recursing into an object I register its clone, so if I encounter the same object again I return the existing clone instead of recursing forever."

---

# 9. Key Takeaways

- Assignment copies the reference; spread copies one level.
- Immutable updates must copy every level on the changed path (structural sharing).
- JSON deep copy: plain data only — know the full list of what breaks.
- `structuredClone`: deep, handles Date/Map/Set/cycles; throws on functions/DOM/symbols; loses prototypes.
- `deepClone` = recursion + special cases + `WeakMap` for cycles (register before recursing).
- `deepEqual` = `Object.is` + prototype check + key comparison + recursion.

## Readiness Checklist

- [ ] Draw shallow vs deep copy memory diagrams
- [ ] List JSON deep copy limitations from memory
- [ ] Explain `structuredClone` support & limits
- [ ] Implement `deepClone` with Date/RegExp/Map/Set/cycles
- [ ] Implement `deepEqual` and `shallowEqual`
- [ ] Write a nested immutable update without mistakes
