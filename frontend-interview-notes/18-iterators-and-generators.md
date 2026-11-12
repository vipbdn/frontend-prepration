# 18 — Iterators & Generators

> **Interview + Understanding Guide** — `00-starter-roadmap.md` Section 18
>
> Goal: Understand the iteration protocols that power `for...of`, spread and destructuring; write custom iterables; and use generators (sync and async) for lazy sequences, pagination and streams.

---

# 1. Why Iteration Protocols?

`for...of`, spread `[...x]`, destructuring `[a, b] = x`, `Array.from(x)`, `new Map(x)`, `Promise.all(x)` all work on **any iterable** — arrays, strings, Maps, Sets, NodeLists, arguments… and **your own objects** if they follow the protocol.

---

# 2. The Iterator Protocol

An **iterator** is an object with a `next()` method that returns `{ value, done }`.

```js
const iterator = {
  current: 1,
  next() {
    if (this.current <= 3) {
      return { value: this.current++, done: false };
    }
    return { value: undefined, done: true };
  },
};

iterator.next(); // { value: 1, done: false }
iterator.next(); // { value: 2, done: false }
iterator.next(); // { value: 3, done: false }
iterator.next(); // { value: undefined, done: true }
```

Optional: `return()` (called on early exit — `break`, `throw`) for cleanup.

---

# 3. The Iterable Protocol

An **iterable** is an object with a `[Symbol.iterator]()` method that **returns an iterator**.

```js
const arr = [10, 20];
const it = arr[Symbol.iterator](); // built-in arrays are iterable
it.next(); // { value: 10, done: false }
it.next(); // { value: 20, done: false }
it.next(); // { value: undefined, done: true }
```

```text
Iterable ── [Symbol.iterator]() ──▶ Iterator ── next() ──▶ { value, done }
```

### What `for...of` does internally

```js
for (const x of iterable) { body(x); }

// ≈
const iterator = iterable[Symbol.iterator]();
let result = iterator.next();
while (!result.done) {
  body(result.value);
  result = iterator.next();
}
```

### Built-in iterables

Array, String (iterates **code points**, so emoji are handled correctly), Map, Set, TypedArray, `arguments`, NodeList, generators.

**Plain objects are NOT iterable:**

```js
for (const x of { a: 1 }) {} // ❌ TypeError: {...} is not iterable
for (const [k, v] of Object.entries({ a: 1 })) {} // ✅
```

### `for...of` vs `for...in`

| `for...of` | `for...in` |
| --- | --- |
| Iterates **values** of an iterable | Iterates **keys** (enumerable string props, incl. inherited) |
| Arrays, strings, Maps, Sets | Objects |
| Respects iteration protocol | Doesn't care about iterators |

```js
for (const i in ["a", "b"]) console.log(i); // "0", "1" (strings!)
for (const v of ["a", "b"]) console.log(v); // "a", "b"
```

---

# 4. Custom Iterable Objects ⭐

```js
class Range {
  constructor(start, end, step = 1) {
    this.start = start;
    this.end = end;
    this.step = step;
  }

  [Symbol.iterator]() {
    let current = this.start;
    const { end, step } = this;
    return {
      next() {
        if (current <= end) {
          const value = current;
          current += step;
          return { value, done: false };
        }
        return { value: undefined, done: true };
      },
    };
  }
}

const range = new Range(1, 5);
[...range];                 // [1, 2, 3, 4, 5]
for (const n of range) {}   // works
const [first, second] = range; // 1, 2
Math.max(...range);         // 5
```

Each call to `[Symbol.iterator]()` returns a **fresh** iterator → the range can be iterated many times.

---

# 5. Generators ⭐

A **generator function** (`function*`) returns a **generator object** — which is both an **iterator and an iterable**. It can **pause** at `yield` and **resume** later.

```js
function* count() {
  console.log("start");
  yield 1;
  console.log("after 1");
  yield 2;
  return 3;
}

const gen = count();   // nothing runs yet! (lazy)
gen.next();            // logs "start"   → { value: 1, done: false }
gen.next();            // logs "after 1" → { value: 2, done: false }
gen.next();            //                → { value: 3, done: true }
gen.next();            //                → { value: undefined, done: true }
```

### Key properties

- Calling a generator function **doesn't run** its body — it returns a paused generator.
- Each `next()` runs until the next `yield` (or `return`).
- `return` value appears with `done: true` — and is **ignored by `for...of` and spread**.

```js
[...count()]; // [1, 2]  — return value 3 not included
```

### Generators make iterables trivial

```js
class Range {
  constructor(start, end) {
    this.start = start;
    this.end = end;
  }
  *[Symbol.iterator]() {
    for (let i = this.start; i <= this.end; i++) yield i;
  }
}
[...new Range(1, 3)]; // [1, 2, 3]
```

---

# 6. Two-Way Communication: `next(value)`, `throw`, `return`

```js
function* conversation() {
  const name = yield "What's your name?";
  const age = yield `Hello ${name}! Age?`;
  return `${name} is ${age}`;
}

const g = conversation();
g.next().value;        // "What's your name?"   (first next() starts it; its arg is ignored)
g.next("Asha").value;  // "Hello Asha! Age?"    ("Asha" becomes the result of the 1st yield)
g.next(25).value;      // "Asha is 25"
```

```js
g.return("bye"); // finishes the generator early → { value: "bye", done: true }
g.throw(new Error("x")); // throws inside the generator at the paused yield
```

This pause/resume + send-values mechanism is how **async/await was originally implemented** (generators + promises, e.g. the `co` library and Babel's regenerator).

---

# 7. `yield*` — Delegation

```js
function* inner() {
  yield "a";
  yield "b";
}

function* outer() {
  yield 1;
  yield* inner();      // yield every value from another iterable/generator
  yield* [2, 3];       // works with any iterable
}

[...outer()]; // [1, "a", "b", 2, 3]
```

### Recursive tree traversal with `yield*`

```js
function* walk(node) {
  yield node.value;
  for (const child of node.children ?? []) {
    yield* walk(child);
  }
}

const tree = { value: 1, children: [{ value: 2 }, { value: 3, children: [{ value: 4 }] }] };
[...walk(tree)]; // [1, 2, 3, 4]
```

### Flatten with a generator

```js
function* flatten(arr) {
  for (const item of arr) {
    if (Array.isArray(item)) yield* flatten(item);
    else yield item;
  }
}
[...flatten([1, [2, [3, [4]]]])]; // [1, 2, 3, 4]
```

---

# 8. Lazy Evaluation & Infinite Sequences ⭐

Generators produce values **on demand** → you can model infinite sequences without infinite memory.

```js
function* naturals() {
  let n = 1;
  while (true) yield n++;
}

function* take(iterable, count) {
  if (count <= 0) return;
  for (const x of iterable) {
    yield x;
    if (--count <= 0) return; // early return closes the source generator
  }
}

function* map(iterable, fn) {
  for (const x of iterable) yield fn(x);
}

function* filter(iterable, pred) {
  for (const x of iterable) if (pred(x)) yield x;
}

const firstFiveEvenSquares = [
  ...take(map(filter(naturals(), (n) => n % 2 === 0), (n) => n * n), 5),
];
// [4, 16, 36, 64, 100] — only computes what's needed
```

> Modern JS also has built-in **iterator helpers** (ES2025): `naturals().filter(...).map(...).take(5).toArray()`.

### Practical: unique ID generator

```js
function* idGenerator(prefix = "id") {
  let i = 0;
  while (true) yield `${prefix}-${++i}`;
}
const nextId = idGenerator("todo");
nextId.next().value; // "todo-1"
nextId.next().value; // "todo-2"
```

---

# 9. Async Generators & `for await...of` ⭐

`async function*` can `await` inside and `yield` values over time. Consumed with `for await...of`.

```js
async function* fetchPages(url) {
  let next = url;
  while (next) {
    const res = await fetch(next);
    const { items, nextPageUrl } = await res.json();
    yield items;
    next = nextPageUrl;
  }
}

for await (const items of fetchPages("/api/products?page=1")) {
  renderProducts(items);
  if (items.length < 20) break; // break calls return() → generator stops → no more fetches
}
```

### Async iterable protocol

Same as sync, but uses `[Symbol.asyncIterator]()` and `next()` returns a **promise** of `{ value, done }`.

### Streams

```js
// Read a fetch response stream chunk by chunk (modern browsers/Node)
const res = await fetch("/big-file.txt");
const decoder = new TextDecoder();
for await (const chunk of res.body) {
  process(decoder.decode(chunk, { stream: true }));
}
```

### `for await` with an array of promises

```js
for await (const result of [fetchA(), fetchB()]) {
  console.log(result); // in array order
}
```

---

# 10. Real-World Uses

| Use | Why generators/iterators |
| --- | --- |
| Custom collections (tree, linked list, range) | Make them work with `for...of`, spread, destructuring |
| Pagination / infinite feeds | Async generator yields pages on demand |
| Streaming data | Process chunks without loading everything |
| Lazy pipelines | Compute only what's consumed |
| Unique IDs, sequences | Infinite generators |
| Redux-Saga | Generators for side effect orchestration (`yield call(api)`) |
| Tree/graph traversal | `yield*` recursion |

---

# 11. Output Questions 🧪

### Q1

```js
function* g() {
  yield 1;
  yield 2;
  return 3;
}
console.log([...g()]);
for (const x of g()) console.log(x);
```

<details><summary>Answer</summary>

`[1, 2]`, then `1`, `2` — the `return` value is ignored by spread and `for...of`.

</details>

### Q2

```js
function* g() {
  const x = yield 1;
  console.log("x =", x);
  yield x * 2;
}
const it = g();
console.log(it.next(5).value);
console.log(it.next(10).value);
```

<details><summary>Answer</summary>

```text
1
x = 10
20
```

The argument to the **first** `next()` is ignored (no `yield` is waiting yet).

</details>

### Q3

```js
function* g() {
  console.log("running");
  yield 1;
}
const it = g();
console.log("created");
it.next();
```

<details><summary>Answer</summary>

`created`, then `running` — generator bodies are lazy.

</details>

### Q4

```js
const it = [1, 2, 3][Symbol.iterator]();
for (const x of it) {
  console.log(x);
  break;
}
for (const x of it) console.log(x);
```

<details><summary>Answer</summary>

`1`, then `2`, `3`. Array iterators don't implement `return()`, so breaking doesn't close them; the second loop continues where the first stopped. (A **generator** would be closed by `break` — the second loop would print nothing.)

</details>

### Q5

```js
const [a, , b] = "hey";
console.log(a, b);
console.log([..."😀a"].length, "😀a".length);
```

<details><summary>Answer</summary>

`h y`, then `2 3` — string iteration goes by code points (emoji = 1), while `.length` counts UTF-16 code units (emoji = 2).

</details>

---

# 12. Interview Answers

**Iterator vs iterable**

> "An iterator is an object with a `next()` method returning `{ value, done }`. An iterable is an object with a `Symbol.iterator` method that returns an iterator. `for...of`, spread and destructuring all work on iterables — arrays, strings, Maps and Sets are built-in, and you can make your own objects iterable by implementing `Symbol.iterator`."

**What are generators?**

> "Generator functions, declared with `function*`, return a generator object that is both an iterator and iterable. They run lazily and pause at each `yield`, resuming on the next `next()` call, which can also send a value in. They're great for custom iterables, lazy or infinite sequences, and, as async generators with `for await...of`, for pagination and streams."

---

# 13. Key Takeaways

- Iterator: `next() → { value, done }`. Iterable: `[Symbol.iterator]() → iterator`.
- `for...of`, spread, destructuring, `Array.from` use the iterable protocol.
- Plain objects aren't iterable — use `Object.entries`.
- Generators: lazy, pausable, both iterator & iterable; `return` value skipped by `for...of`.
- `next(value)` sends values in; first `next` argument ignored.
- `yield*` delegates to another iterable — great for recursion.
- Async generators + `for await...of` for pagination and streams.

## Readiness Checklist

- [ ] Write a custom iterable with and without a generator
- [ ] Explain what `for...of` does internally
- [ ] Trace a generator with `next(value)`
- [ ] Build lazy `take/map/filter` with generators
- [ ] Write an async generator for paginated APIs
