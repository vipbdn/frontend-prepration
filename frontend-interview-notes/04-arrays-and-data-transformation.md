# 04 — Arrays & Data Transformation

> **Interview + Understanding Guide** — `00-starter-roadmap.md` Section 4
>
> Goal: Know every common array method (what it returns, whether it mutates), and solve the data-transformation problems that show up in frontend interviews.

---

# 1. The Most Important Table ⭐⭐⭐

| Method        | Returns                          | Mutates? | Callback args              |
| ------------- | -------------------------------- | -------- | -------------------------- |
| `map`         | New array (same length)          | ❌       | `(item, index, array)`     |
| `filter`      | New array (subset)               | ❌       | `(item, index, array)`     |
| `reduce`      | Any single value                 | ❌       | `(acc, item, index, array)`|
| `forEach`     | `undefined`                      | ❌*      | `(item, index, array)`     |
| `find`        | First matching item / `undefined`| ❌       |                            |
| `findIndex`   | Index / `-1`                     | ❌       |                            |
| `findLast` / `findLastIndex` | From the end (ES2023) | ❌  |                            |
| `some`        | `true` if **any** match          | ❌       |                            |
| `every`       | `true` if **all** match          | ❌       |                            |
| `includes`    | Boolean (uses SameValueZero → finds `NaN`) | ❌ |                    |
| `indexOf`     | Index / `-1` (uses `===` → can't find `NaN`) | ❌ |                  |
| `sort`        | **Same** array, sorted           | ✅       | `(a, b)`                   |
| `reverse`     | **Same** array, reversed         | ✅       |                            |
| `splice`      | Array of **removed** items       | ✅       |                            |
| `push` / `unshift` | New **length**              | ✅       |                            |
| `pop` / `shift` | Removed item                   | ✅       |                            |
| `fill`        | Same array                       | ✅       |                            |
| `slice`       | New array (portion)              | ❌       |                            |
| `concat`      | New array                        | ❌       |                            |
| `join`        | String                           | ❌       |                            |
| `flat`        | New array                        | ❌       |                            |
| `flatMap`     | New array (map then flat 1 level)| ❌       |                            |
| `toSorted` / `toReversed` / `toSpliced` / `with` | New array (ES2023) | ❌ |      |

\* `forEach` itself doesn't mutate, but your callback can.

> **Memory trick:** `sort`, `reverse`, `splice`, `push/pop/shift/unshift`, `fill`, `copyWithin` mutate. Everything else returns something new.

---

# 2. `map`, `filter`, `reduce`, `forEach`

### `map` — transform each item

```js
const users = [{ name: "Asha", age: 25 }, { name: "Ravi", age: 17 }];
const names = users.map((u) => u.name); // ["Asha", "Ravi"]
```

⚠️ Returning an object literal from an arrow needs parentheses:

```js
users.map((u) => ({ label: u.name })); // ✅
users.map((u) => { label: u.name });   // ❌ [undefined, undefined] — treated as a block with a label
```

### `filter` — keep items where callback is truthy

```js
const adults = users.filter((u) => u.age >= 18);

// Remove falsy values
[0, "a", null, "b", undefined, NaN].filter(Boolean); // ["a", "b"]
```

### `reduce` — boil an array down to one value

```js
const total = [10, 20, 30].reduce((sum, n) => sum + n, 0); // 60
```

```text
acc=0,  n=10 → 10
acc=10, n=20 → 30
acc=30, n=30 → 60
```

**Always pass an initial value.** Without it:
- First element is used as initial `acc` (iteration starts at index 1)
- Empty array → `TypeError: Reduce of empty array with no initial value`

`reduce` can build **anything**:

```js
// Array → object (lookup by id)
const byId = users.reduce((acc, u) => {
  acc[u.name] = u;
  return acc; // ⚠️ forgetting `return acc` is the #1 reduce bug
}, {});

// Count occurrences
const counts = ["a", "b", "a"].reduce((acc, ch) => {
  acc[ch] = (acc[ch] || 0) + 1;
  return acc;
}, {}); // { a: 2, b: 1 }
```

### `forEach`

- Returns `undefined` → can't chain
- **Can't `break`** out early (use `for...of`, `some`, `every`, or `find`)
- **Doesn't wait for `async` callbacks** (see `07-async-javascript.md`)

```js
[1, 2, 3].forEach(async (n) => { await save(n); });
console.log("done"); // logs BEFORE any save finishes ❗
```

### Sparse arrays & holes

`map/filter/forEach/reduce` **skip holes**:

```js
const arr = [1, , 3];
arr.map((x) => x * 2); // [2, <empty>, 6]
new Array(3).map(() => 0); // [ <3 empty items> ] ❗
Array.from({ length: 3 }, () => 0); // [0, 0, 0] ✅
```

---

# 3. Searching

```js
const nums = [5, 12, 8, 130, 44];

nums.find((n) => n > 10);      // 12 (first match)
nums.findIndex((n) => n > 10); // 1
nums.findLast((n) => n > 10);  // 44
nums.some((n) => n > 100);     // true — stops at first true
nums.every((n) => n > 1);      // true — stops at first false
nums.includes(8);              // true
nums.indexOf(8);               // 2

[NaN].includes(NaN); // true
[NaN].indexOf(NaN);  // -1 ❗
[].every(() => false); // true ❗ (vacuous truth)
[].some(() => true);   // false
```

---

# 4. `sort` — The Trap-Filled One ⭐

```js
[10, 1, 2].sort();               // [1, 10, 2] ❗ default = string comparison
[10, 1, 2].sort((a, b) => a - b); // [1, 2, 10] ascending
[10, 1, 2].sort((a, b) => b - a); // [10, 2, 1] descending
```

Comparator rule: return **negative** → `a` first, **positive** → `b` first, **0** → keep order.

```js
// Strings, locale-aware
names.sort((a, b) => a.localeCompare(b));

// Objects by multiple keys
users.sort((a, b) => a.age - b.age || a.name.localeCompare(b.name));
```

- `sort` is **stable** (equal items keep original order) since ES2019.
- It **mutates** — in React state, use `[...arr].sort()` or `arr.toSorted()`.
- `undefined` values always go to the end.

---

# 5. `slice` vs `splice` ⭐

| | `slice(start, end)` | `splice(start, deleteCount, ...items)` |
|---|---|---|
| Mutates | ❌ | ✅ |
| Returns | Copied portion (end exclusive) | Removed elements |
| Use | Copy / take part | Insert / remove / replace in place |

```js
const a = [1, 2, 3, 4, 5];
a.slice(1, 3);  // [2, 3] — a unchanged
a.slice(-2);    // [4, 5]
a.slice();      // shallow copy

const b = [1, 2, 3, 4, 5];
b.splice(1, 2);        // returns [2, 3]; b = [1, 4, 5]
b.splice(1, 0, "x");   // insert; b = [1, "x", 4, 5]
```

Immutable alternatives: `b.toSpliced(1, 2)`, or `[...b.slice(0, i), item, ...b.slice(i)]`.

---

# 6. Other Useful Methods

```js
[1, 2].concat([3], 4);           // [1, 2, 3, 4]
["a", "b"].join("-");            // "a-b"
[1, [2, [3, [4]]]].flat();       // [1, 2, [3, [4]]] — default depth 1
[1, [2, [3, [4]]]].flat(Infinity); // [1, 2, 3, 4]
["a b", "c"].flatMap((s) => s.split(" ")); // ["a", "b", "c"]
new Array(3).fill(0);            // [0, 0, 0]

Array.from("abc");                       // ["a", "b", "c"]
Array.from({ length: 3 }, (_, i) => i);  // [0, 1, 2]
Array.from(new Set([1, 1, 2]));          // [1, 2]
Array.from(document.querySelectorAll("li")); // NodeList → real array

Array.of(3);    // [3]
new Array(3);   // [ <3 empty items> ] ❗ single number = length
```

### `fill` with objects trap

```js
const grid = new Array(3).fill([]);
grid[0].push(1);
console.log(grid); // [[1], [1], [1]] ❗ same array reference three times

const fixed = Array.from({ length: 3 }, () => []); // ✅ separate arrays
```

### Array checks

```js
Array.isArray([]);   // true ✅
typeof [];           // "object" ❗
```

---

# 7. Data Transformation Problems ⭐

### 7.1 Flatten array (implement `flat`)

```js
// Recursive with depth
function flatten(arr, depth = Infinity) {
  return arr.reduce((acc, item) => {
    if (Array.isArray(item) && depth > 0) {
      acc.push(...flatten(item, depth - 1));
    } else {
      acc.push(item);
    }
    return acc;
  }, []);
}

flatten([1, [2, [3, [4]]]]);    // [1, 2, 3, 4]
flatten([1, [2, [3, [4]]]], 1); // [1, 2, [3, [4]]]
```

Iterative (no recursion → no stack overflow on very deep arrays):

```js
function flattenIterative(arr) {
  const stack = [...arr];
  const result = [];
  while (stack.length) {
    const item = stack.pop();
    if (Array.isArray(item)) stack.push(...item);
    else result.push(item);
  }
  return result.reverse();
}
```

### 7.2 Flatten nested object ⭐

```js
function flattenObject(obj, prefix = "", result = {}) {
  for (const [key, value] of Object.entries(obj)) {
    const path = prefix ? `${prefix}.${key}` : key;
    if (value && typeof value === "object" && !Array.isArray(value)) {
      flattenObject(value, path, result);
    } else {
      result[path] = value;
    }
  }
  return result;
}

flattenObject({ a: 1, b: { c: 2, d: { e: 3 } } });
// { a: 1, "b.c": 2, "b.d.e": 3 }
```

Unflatten (reverse) lives in `21-utility-implementations.md`.

### 7.3 Group by property ⭐

```js
function groupBy(arr, keyOrFn) {
  const getKey = typeof keyOrFn === "function" ? keyOrFn : (item) => item[keyOrFn];
  return arr.reduce((acc, item) => {
    const key = getKey(item);
    (acc[key] ||= []).push(item);
    return acc;
  }, {});
}

const people = [
  { name: "Asha", dept: "eng" },
  { name: "Ravi", dept: "design" },
  { name: "Neha", dept: "eng" },
];
groupBy(people, "dept");
// { eng: [Asha, Neha], design: [Ravi] }
groupBy([1.2, 1.8, 2.1], Math.floor); // { 1: [1.2, 1.8], 2: [2.1] }
```

Built-in (ES2024): `Object.groupBy(people, (p) => p.dept)` and `Map.groupBy(...)`.

### 7.4 Remove duplicates ⭐

```js
const unique = (arr) => [...new Set(arr)];
unique([1, 2, 2, 3, 1]); // [1, 2, 3]

// Without Set
const unique2 = (arr) => arr.filter((x, i) => arr.indexOf(x) === i); // O(n²)

// Objects by key
function uniqueBy(arr, key) {
  const seen = new Set();
  return arr.filter((item) => {
    if (seen.has(item[key])) return false;
    seen.add(item[key]);
    return true;
  });
}
```

### 7.5 Find duplicates

```js
function findDuplicates(arr) {
  const seen = new Set();
  const dupes = new Set();
  for (const x of arr) {
    if (seen.has(x)) dupes.add(x);
    else seen.add(x);
  }
  return [...dupes];
}
findDuplicates([1, 2, 3, 2, 1, 1]); // [2, 1]
```

### 7.6 Array ↔ Object

```js
// Array → object keyed by id
const toMap = (arr, key) => Object.fromEntries(arr.map((x) => [x[key], x]));

// Object → array
const obj = { a: 1, b: 2 };
Object.entries(obj).map(([key, value]) => ({ key, value }));
// [{ key: "a", value: 1 }, { key: "b", value: 2 }]
```

### 7.7 Chunk array

```js
function chunk(arr, size) {
  const result = [];
  for (let i = 0; i < arr.length; i += size) {
    result.push(arr.slice(i, i + size));
  }
  return result;
}
chunk([1, 2, 3, 4, 5], 2); // [[1, 2], [3, 4], [5]]
```

### 7.8 Intersection & Difference

```js
const intersection = (a, b) => {
  const setB = new Set(b);
  return [...new Set(a)].filter((x) => setB.has(x));
};

const difference = (a, b) => {
  const setB = new Set(b);
  return a.filter((x) => !setB.has(x));
};

intersection([1, 2, 3], [2, 3, 4]); // [2, 3]
difference([1, 2, 3], [2, 3, 4]);   // [1]
```

Using a `Set` makes lookups O(1) → overall O(n + m) instead of O(n × m). **Mention complexity in interviews.**

---

# 8. Performance Notes

- Chaining `filter().map()` loops twice — fine for small arrays; use one `reduce` or `for` loop for huge ones.
- `for` / `for...of` are fastest and support `break`/`await`.
- `arr.includes` inside a loop = O(n²). Convert to a `Set` first.
- `[...acc, item]` inside `reduce` is O(n²) (copies every time). Use `acc.push(item)` when the accumulator is local.

---

# 9. Output Questions 🧪

```js
console.log([1, 2, 3].map((x) => { x * 2 }));                 // Q1
console.log([1, 2, 3].map(parseInt));                         // Q2
console.log(["1", "2", "3"].map(parseInt));                   // Q3
console.log([3, 20, 100].sort());                             // Q4
console.log([1, 2, 3].push(4));                               // Q5
console.log([0, 1, "", null, "a"].filter(Boolean));           // Q6
console.log([].every((x) => x > 10));                         // Q7
console.log([1, 2, 3].includes("2"));                         // Q8
console.log(Array(3).join("-"));                              // Q9
const arr = [1, 2, 3]; arr.length = 0; console.log(arr);      // Q10
```

<details><summary>Answers</summary>

1. `[undefined, undefined, undefined]` — braces make a block; nothing returned.
2. `[1, NaN, NaN]` — same radix trap: `parseInt(1, 0)`, `parseInt(2, 1)`, `parseInt(3, 2)`.
3. `[1, NaN, NaN]`
4. `[100, 20, 3]` — string comparison: "100" < "20" < "3".
5. `4` — `push` returns the new length.
6. `[1, "a"]`
7. `true` — vacuous truth on empty array.
8. `false` — strict comparison, no coercion.
9. `"--"` — 3 empty slots → 2 separators.
10. `[]` — setting `length` truncates the array.

</details>

---

# 10. Interview Answers

**map vs forEach**

> "`map` returns a new array of the same length with each item transformed, so it's for transformations and can be chained. `forEach` returns `undefined` and is only for side effects. Neither can be broken out of early, and neither awaits async callbacks."

**When to use reduce**

> "When I need to turn an array into a single value of any shape — a sum, an object lookup, a grouped map. I always pass an initial value, and if it gets complicated I prefer a simple loop for readability."

**Which methods mutate?**

> "`sort`, `reverse`, `splice`, `push`, `pop`, `shift`, `unshift`, `fill`. In React state I avoid them or use the ES2023 non-mutating versions like `toSorted` and `toSpliced`."

---

# 11. Key Takeaways

- Know return value + mutation for every method.
- Always give `reduce` an initial value and `return acc`.
- Default `sort` compares strings; it mutates.
- `includes` finds `NaN`; `indexOf` doesn't.
- `forEach` can't break and doesn't await.
- Use `Set` for O(1) lookups in unique/intersection/difference.
- `fill([])` / `fill({})` shares one reference.

## Readiness Checklist

- [ ] Recite which methods mutate
- [ ] Implement flatten (recursive + depth), groupBy, chunk, unique, intersection, difference, flattenObject
- [ ] Explain time complexity of each
- [ ] Solve all output questions
