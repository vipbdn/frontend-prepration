# 17 — Map / Set / WeakMap / WeakSet

> **Interview + Understanding Guide** — `00-starter-roadmap.md` Section 17
>
> Goal: Know when to use `Map` over objects and `Set` over arrays, understand weak references and garbage collection, and use them for caching and private data.

---

# 1. `Map` ⭐

A **key → value** collection where keys can be **any type** (objects, functions, `NaN`…) and **insertion order** is preserved.

```js
const map = new Map();

map.set("name", "Asha");
map.set(42, "number key");
const objKey = { id: 1 };
map.set(objKey, "object key");

map.get("name");     // "Asha"
map.get(objKey);     // "object key"
map.get({ id: 1 });  // undefined — different object reference
map.has(42);         // true
map.delete(42);      // true
map.size;            // 2
map.clear();
```

### Creating & converting

```js
const m = new Map([
  ["a", 1],
  ["b", 2],
]);

const fromObj = new Map(Object.entries({ x: 1, y: 2 }));
const toObj = Object.fromEntries(m);   // { a: 1, b: 2 }
const toArray = [...m];                // [["a", 1], ["b", 2]]
```

### Iteration (in insertion order)

```js
for (const [key, value] of m) console.log(key, value);
m.forEach((value, key) => console.log(key, value)); // note: value first
[...m.keys()];    // ["a", "b"]
[...m.values()];  // [1, 2]
[...m.entries()]; // [["a", 1], ["b", 2]]
```

### Chaining

```js
const m2 = new Map().set("a", 1).set("b", 2); // set returns the map
```

### Key equality: SameValueZero

```js
const m3 = new Map();
m3.set(NaN, "nan");
m3.get(NaN);  // "nan" ✅ (NaN equals NaN here)
m3.set(0, "zero");
m3.get(-0);   // "zero" (+0 and -0 are the same key)
```

---

# 2. Map vs Object ⭐⭐⭐

| | `Map` | Object `{}` |
| --- | --- | --- |
| Key types | **Any** (objects, functions, numbers stay numbers) | Strings & symbols only (others converted to strings) |
| Order | Insertion order, always | Integer keys first (ascending), then insertion order |
| Size | `map.size` (O(1)) | `Object.keys(obj).length` (O(n)) |
| Iteration | Directly iterable (`for...of`) | Need `Object.keys/entries` |
| Prototype keys | None — no collisions | Inherits `toString`, `constructor`… (unless `Object.create(null)`) |
| Performance | Optimized for **frequent add/delete** | Optimized for fixed-shape records |
| JSON | ❌ Not directly serializable | ✅ `JSON.stringify` |
| Syntax | `.get/.set` | `obj.key`, destructuring, spread |

### When to use which

- **Object** → fixed-shape **records/data** (`{ id, name, email }`), JSON, config.
- **Map** → dynamic **dictionaries/lookups**, non-string keys, frequent insertions/deletions, need order or size, caches.

```js
// Object key collision trap
const counts = {};
for (const w of ["constructor", "a"]) {
  counts[w] = (counts[w] || 0) + 1;
}
counts.constructor; // "function Object() { [native code] }1" ❗

const counts2 = new Map();
for (const w of ["constructor", "a"]) {
  counts2.set(w, (counts2.get(w) || 0) + 1);
}
counts2.get("constructor"); // 1 ✅
```

---

# 3. `Set` ⭐

A collection of **unique values** (SameValueZero equality), in insertion order.

```js
const set = new Set([1, 2, 2, 3]);
set.size;        // 3
set.add(4);      // returns the set (chainable)
set.add(2);      // ignored — already present
set.has(2);      // true — O(1)
set.delete(1);
[...set];        // [2, 3, 4]

for (const v of set) console.log(v);
```

### Objects are compared by reference

```js
const s = new Set();
s.add({ a: 1 });
s.add({ a: 1 });
s.size; // 2 — different objects
```

### Common uses

```js
// Remove duplicates
const unique = [...new Set([1, 1, 2, 3, 3])]; // [1, 2, 3]

// Unique characters
new Set("hello").size; // 4

// Fast membership checks
const allowed = new Set(["admin", "editor"]);
if (allowed.has(user.role)) { /* ... */ }

// Track visited items (graphs, deduping events)
const seen = new Set();
```

### Set operations

```js
const a = new Set([1, 2, 3]);
const b = new Set([2, 3, 4]);

const union = new Set([...a, ...b]);                  // {1, 2, 3, 4}
const intersection = new Set([...a].filter((x) => b.has(x))); // {2, 3}
const difference = new Set([...a].filter((x) => !b.has(x)));  // {1}

// Modern built-ins (ES2025, supported in current browsers/Node 22+)
a.union(b); a.intersection(b); a.difference(b);
a.symmetricDifference(b); a.isSubsetOf(b);
```

---

# 4. Set vs Array ⭐

| | `Set` | Array |
| --- | --- | --- |
| Duplicates | ❌ Unique only | ✅ Allowed |
| Lookup (`has` / `includes`) | **O(1)** | O(n) |
| Index access | ❌ | ✅ `arr[0]` |
| Order | Insertion | Index |
| Methods | `add/has/delete` + set ops | `map/filter/reduce/sort`… |
| JSON | ❌ | ✅ |

Use a `Set` when you care about **uniqueness** or **fast membership checks**; convert to an array for transformations.

```js
// O(n²) → O(n)
const blockedIds = new Set(blocked.map((b) => b.id));
const visible = posts.filter((p) => !blockedIds.has(p.authorId));
```

---

# 5. `WeakMap` ⭐

Like a `Map`, but:

1. **Keys must be objects** (or non-registered symbols) — no primitives
2. Keys are held **weakly** — if nothing else references the key object, the entry can be **garbage-collected**
3. **Not iterable**, no `.size`, no `.clear()` — because contents depend on GC timing

```js
const wm = new WeakMap();
let user = { name: "Asha" };

wm.set(user, { visits: 1 });
wm.get(user); // { visits: 1 }

user = null;
// The { name: "Asha" } object is now unreachable → the entry disappears at the next GC.
// With a normal Map, the map itself would keep the object alive forever (leak).
```

API: `set`, `get`, `has`, `delete` — that's it.

## 5.1 Weak References & GC

```text
Map:      map ──strong──▶ key object   → key can't be collected while map exists
WeakMap:  wm  ┄┄weak┄┄┄▶ key object   → doesn't count for reachability
```

> The **value** is held strongly **as long as the key is alive**. Once the key is collected, the value becomes collectible too (if nothing else references it).

## 5.2 Use case — caching per object

```js
const cache = new WeakMap();

function computeStats(data) {
  if (cache.has(data)) return cache.get(data);
  const result = expensive(data);
  cache.set(data, result);
  return result;
}
```

When `data` is no longer used anywhere, its cache entry vanishes automatically — **no memory leak**, no manual cleanup.

## 5.3 Use case — private data

```js
const privateData = new WeakMap();

class BankAccount {
  constructor(balance) {
    privateData.set(this, { balance }); // not accessible from outside the module
  }
  deposit(amount) {
    privateData.get(this).balance += amount;
  }
  get balance() {
    return privateData.get(this).balance;
  }
}

const acc = new BankAccount(100);
acc.deposit(50);
acc.balance;  // 150
acc.balance = 0; // no setter → ignored
// Nothing like acc._balance to tamper with
```

(Modern alternative: `#private` class fields. The WeakMap pattern is how Babel compiles them.)

## 5.4 Use case — metadata on DOM nodes

```js
const elementState = new WeakMap();

function track(el) {
  elementState.set(el, { clicks: 0 });
}
// When the element is removed from the DOM and dropped from JS, its state is collected too.
```

---

# 6. `WeakSet`

Like a `Set`, but values must be objects, held weakly, not iterable.

API: `add`, `has`, `delete`.

```js
const processed = new WeakSet();

function processOnce(obj) {
  if (processed.has(obj)) return;
  processed.add(obj);
  doWork(obj);
}
```

Use cases:
- "Have I seen this object?" without preventing GC
- Marking objects (e.g., branded instances, visited nodes in a deep clone/traversal)
- Tracking circular references

```js
const instances = new WeakSet();
class Token {
  constructor() { instances.add(this); }
  static isToken(x) { return instances.has(x); } // brand check, unforgeable
}
```

---

# 7. Comparison Table ⭐

| | `Map` | `WeakMap` | `Set` | `WeakSet` |
| --- | --- | --- | --- | --- |
| Stores | key → value | object key → value | unique values | unique objects |
| Key/value types | Any | Keys: objects only | Any | Objects only |
| Holds references | Strong | **Weak** (keys) | Strong | **Weak** |
| Iterable | ✅ | ❌ | ✅ | ❌ |
| `.size` | ✅ | ❌ | ✅ | ❌ |
| `.clear()` | ✅ | ❌ | ✅ | ❌ |
| Prevents GC | ✅ | ❌ | ✅ | ❌ |
| Typical use | Dictionaries, caches, lookups | Per-object caches, private data, DOM metadata | Uniqueness, fast membership | Marking/tracking objects |

---

# 8. Caching with Map

```js
// Simple memo cache (unbounded — be careful)
const cache = new Map();
function getUser(id) {
  if (cache.has(id)) return cache.get(id);
  const promise = fetch(`/api/users/${id}`).then((r) => r.json());
  cache.set(id, promise);
  return promise;
}
```

- Primitive keys (ids, strings) → `Map` (with LRU/TTL limits to avoid leaks)
- Object keys → `WeakMap` (auto cleanup)

LRU cache implementation using `Map` insertion order: see `10-memoization-and-caching.md` §4.1.

---

# 9. Output Questions 🧪

```js
const m = new Map();
m.set("1", "str");
m.set(1, "num");
console.log(m.size, m.get(1), m.get("1"));                // Q1

const o = {};
o["1"] = "str";
o[1] = "num";
console.log(Object.keys(o).length, o["1"]);               // Q2

const s = new Set([NaN, NaN, 0, -0, "0"]);
console.log(s.size);                                       // Q3

const s2 = new Set([[1], [1]]);
console.log(s2.size);                                      // Q4

const wm = new WeakMap();
try { wm.set("key", 1); } catch (e) { console.log(e.name); } // Q5

console.log(JSON.stringify(new Map([["a", 1]])));          // Q6

console.log([...new Set("mississippi")].join(""));         // Q7
```

<details><summary>Answers</summary>

1. `2 num str` — Map keeps `"1"` and `1` distinct.
2. `1 num` — object keys are strings; `1` → `"1"` overwrites.
3. `3` — `NaN` once, `0`/`-0` same, `"0"` separate.
4. `2` — two different array references.
5. `TypeError` — WeakMap keys must be objects.
6. `{}` — Maps don't serialize; use `Object.fromEntries` or `[...map]`.
7. `"misp"` — unique characters in first-seen order.

</details>

---

# 10. Interview Answers

**Map vs Object**

> "I use objects for fixed-shape records and JSON data. I use `Map` for dynamic key–value collections: keys can be any type, it preserves insertion order, has a `size` property, is directly iterable, has no prototype key collisions, and performs better with frequent additions and deletions."

**What is a WeakMap and why use it?**

> "A `WeakMap` only accepts objects as keys and holds them weakly — it doesn't prevent garbage collection. When the key object is no longer referenced elsewhere, its entry disappears. That makes it ideal for caching computed data per object, storing private data for instances, or attaching metadata to DOM nodes without causing memory leaks. The trade-off is that it's not iterable and has no size."

---

# 11. Key Takeaways

- `Map`: any key type, ordered, `size`, iterable — for dynamic lookups.
- `Set`: unique values, O(1) `has` — dedupe & membership.
- Both use SameValueZero (`NaN` equals `NaN`; objects by reference).
- `WeakMap`/`WeakSet`: object keys, weak references, not iterable — no leaks.
- Use `WeakMap` for per-object caches, private data, DOM metadata.
- Maps/Sets don't JSON-serialize directly.

## Readiness Checklist

- [ ] Explain 5 differences between Map and Object
- [ ] Use Set for dedupe, intersection, difference
- [ ] Explain weak references and GC with a diagram
- [ ] Implement a WeakMap cache and WeakMap private data
- [ ] Solve all output questions
