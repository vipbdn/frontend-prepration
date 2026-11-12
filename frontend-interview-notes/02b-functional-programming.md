# JavaScript Functional Programming — Complete Interview & Revision Notes

> **Interview + Understanding Guide** — Section 2 of `00-starter-roadmap.md` (Closures & Functional JavaScript)
>
> Goal: Understand functional programming concepts deeply enough to explain them, implement the utilities from scratch, solve output questions, and connect everything to React.
>
> **Prerequisite:** Closures (`02a-closures.md`). Almost every utility in this file is a closure.

---

## Roadmap Coverage

| `00-starter-roadmap.md` item           | Covered in       |
| --------------------------- | ---------------- |
| Higher-order functions      | Section 2        |
| Pure functions              | Section 3        |
| Side effects                | Section 4        |
| Immutability (supporting)   | Section 5        |
| Function factory            | Section 6        |
| Function composition        | Section 7        |
| `compose` / `pipe`          | Sections 7, 8    |
| Currying / `curry()`        | Sections 9, 10   |
| Partial application         | Section 11       |
| `once()`                    | Section 12       |
| FP in React                 | Section 14       |
| Output questions + practice | Sections 15–17   |

> Debounce, throttle and memoize are also higher-order functions, but `00-starter-roadmap.md` gives them their own sections (9 and 10). They are only previewed here.

---

# 1. What is Functional Programming?

**Functional Programming (FP)** is a style of writing code where you build programs by **combining small, pure functions** and avoid **shared mutable state** and **side effects** as much as possible.

### Interview Definition

> **Functional programming is a programming paradigm that treats functions as the main building blocks. It favors pure functions, immutable data, and composing small functions together, instead of changing shared state step by step.**

JavaScript is **not** a purely functional language (like Haskell). It is multi-paradigm. But it supports FP well because **functions are first-class citizens**.

### Imperative vs Declarative

```js
// Imperative — HOW to do it, step by step
const nums = [1, 2, 3, 4];
const result = [];
for (let i = 0; i < nums.length; i++) {
  if (nums[i] % 2 === 0) {
    result.push(nums[i] * 10);
  }
}

// Declarative / functional — WHAT you want
const result2 = nums.filter((n) => n % 2 === 0).map((n) => n * 10);
```

Both give `[20, 40]`. The functional version:

- Does not mutate anything
- Reads like a description of the result
- Is built from small reusable pieces

### Core Pillars

```text
Functional Programming
│
├── First-class functions
├── Higher-order functions
├── Pure functions
├── No / isolated side effects
├── Immutability
└── Composition (compose / pipe / currying)
```

---

## 1.1 First-Class Functions (the foundation)

In JavaScript, functions are **values**. You can:

```js
// 1. Store them in variables
const greet = function () {
  return "Hi";
};

// 2. Store them in objects / arrays
const handlers = { onClick: () => console.log("clicked") };

// 3. Pass them as arguments
setTimeout(greet, 1000);

// 4. Return them from other functions
function makeGreeter() {
  return () => "Hello";
}
```

> **First-class functions** is a property of the **language**.
> **Higher-order function** is a property of a **specific function**.
>
> Because JS has first-class functions, we can write higher-order functions.

---

# 2. Higher-Order Functions (HOF)

## What is a Higher-Order Function?

A **higher-order function** is a function that does at least one of these:

1. **Takes a function as an argument**, or
2. **Returns a function**

```text
Higher-Order Function
│
├── accepts a function   → map, filter, reduce, setTimeout, addEventListener
└── returns a function   → function factories, curry, debounce, bind
```

### Takes a function

```js
function repeat(n, action) {
  for (let i = 0; i < n; i++) {
    action(i);
  }
}

repeat(3, console.log);
```

Output:

```text
0
1
2
```

`repeat` is the HOF. `console.log` is the **callback**.

### Returns a function

```js
function multiplier(factor) {
  return function (n) {
    return n * factor;
  };
}

const double = multiplier(2);
console.log(double(5));
```

Output:

```text
10
```

`multiplier` returns a function → it's a HOF. The returned function is a **closure** over `factor`.

### Built-in HOFs You Use Daily

| HOF                     | Takes a function? | Returns a function? |
| ----------------------- | ----------------- | ------------------- |
| `Array.prototype.map`   | ✅                | ❌                  |
| `filter`, `reduce`      | ✅                | ❌                  |
| `sort(compareFn)`       | ✅                | ❌                  |
| `setTimeout`            | ✅                | ❌                  |
| `addEventListener`      | ✅                | ❌                  |
| `Promise.then`          | ✅                | ❌                  |
| `Function.prototype.bind` | ❌ (called on one) | ✅              |
| `debounce`, `throttle`  | ✅                | ✅                  |

### Why HOFs Matter

- **Abstraction** — hide the loop/timing/caching logic, expose only the "what"
- **Reusability** — one `debounce` works for any function
- **Composition** — small functions combine into bigger ones

### Preview: utilities that are HOFs

```js
debounce(fn, delay); // returns a delayed version of fn      → starter.md §9
throttle(fn, limit); // returns a rate-limited version of fn → starter.md §9
memoize(fn);         // returns a caching version of fn      → starter.md §10
once(fn);            // returns a run-only-once version      → Section 12 below
```

> Pattern to notice: **take a function, return an enhanced function**. This is called a **function decorator / wrapper**. React's Higher-Order Components use the same idea with components.

## Interview Answer

> "A higher-order function is a function that takes another function as an argument, returns a function, or both. JavaScript allows this because functions are first-class values. Examples are `map`, `filter`, `reduce`, `setTimeout`, and utilities like `debounce` or `memoize` that take a function and return an enhanced version of it."

## Key Takeaways

- HOF = takes a function **or** returns a function.
- Possible because functions are first-class in JS.
- Callbacks are the functions passed **into** a HOF.
- "Take fn → return enhanced fn" is the decorator pattern behind debounce, throttle, memoize, once.

---

# 3. Pure Functions

## What is a Pure Function?

A function is **pure** when:

1. **Same input → always the same output** (deterministic)
2. **No side effects** (doesn't change or depend on anything outside itself)

```js
// ✅ Pure
function add(a, b) {
  return a + b;
}

add(2, 3); // 5 — always 5, forever
```

### Impure — depends on external state

```js
let tax = 0.18;

function priceWithTax(price) {
  return price + price * tax; // reads outer mutable variable
}

priceWithTax(100); // 118
tax = 0.28;
priceWithTax(100); // 128 — same input, different output ❌
```

**Fix:** pass it in.

```js
function priceWithTax(price, tax) {
  return price + price * tax;
}
```

### Impure — mutates its input

```js
function addItem(cart, item) {
  cart.push(item); // ❌ mutates the caller's array
  return cart;
}
```

**Fix:** return a new value.

```js
function addItem(cart, item) {
  return [...cart, item]; // ✅ original untouched
}
```

### Impure — non-deterministic

```js
function getId() {
  return Math.random(); // different every call
}

function greeting() {
  return `Today is ${new Date().toDateString()}`; // depends on the clock
}
```

### Quick Purity Test

Ask two questions:

```text
1. If I call it again with the same arguments, do I ALWAYS get the same result?
2. Can I delete the call (and keep only its return value) without changing anything else in the program?

Both YES → pure
```

Question 2 is called **referential transparency**: a pure call `add(2, 3)` can be replaced by `5` anywhere and the program behaves the same.

### Common Hidden Impurity

| Looks harmless                        | Why it's impure             |
| ------------------------------------- | --------------------------- |
| `arr.sort()` / `arr.reverse()`        | Mutates the array in place  |
| `arr.push()` / `splice()`             | Mutates                     |
| `obj.count++` on an argument          | Mutates the caller's object |
| `console.log()`                       | I/O side effect             |
| `Math.random()`, `Date.now()`         | Non-deterministic           |
| Reading a global / module variable that changes | Hidden input      |
| `fetch()`, `localStorage.setItem()`   | Network / storage I/O       |

### Why Pure Functions Matter

- **Easy to test** — no setup/mocks, just input → expected output
- **Predictable** — no "it works sometimes" bugs
- **Cacheable** — safe to memoize (same input = same output)
- **Safe to run in parallel / re-run** — React relies on this
- **Easy to reason about** — everything the function needs is in its parameters

## Interview Answer

> "A pure function always returns the same output for the same input and has no side effects — it doesn't mutate its arguments, change outside state, or do I/O. That makes it predictable, easy to test, and safe to memoize. In React, components and reducers are expected to be pure."

## Key Takeaways

- Pure = deterministic + no side effects.
- Hidden inputs (globals, time, randomness) break purity.
- Mutating arguments breaks purity — return new values instead.
- Purity is what makes memoization and React rendering safe.

---

# 4. Side Effects

## What is a Side Effect?

A **side effect** is anything a function does **besides computing and returning a value** — any interaction with the world outside the function.

```text
Side effects
│
├── Mutating arguments or outer variables
├── Network requests (fetch, axios)
├── DOM manipulation
├── Writing to localStorage / cookies
├── Logging (console.log)
├── Timers (setTimeout, setInterval)
├── Subscriptions / event listeners
└── Throwing / reading random or time values (non-determinism)
```

### Side effects are NOT bad

An app with **zero** side effects does nothing useful — it can't show anything, save anything, or call an API.

> The FP goal is not "no side effects". It is **"keep side effects isolated, explicit, and at the edges"** while keeping the core logic pure.

```text
           ┌──────────────────────────┐
 input ──▶ │   Pure core logic        │ ──▶ result
           │ (calculate, transform)   │
           └──────────────────────────┘
                       │
              Impure shell (edges)
       fetch · DOM · storage · logging
```

### Example — Separate the pure part from the effect

```js
// ❌ Mixed: calculation + DOM + storage in one function
function checkout(cart) {
  const total = cart.reduce((sum, item) => sum + item.price * item.qty, 0);
  document.querySelector("#total").textContent = total;
  localStorage.setItem("lastTotal", total);
}

// ✅ Pure core
function calculateTotal(cart) {
  return cart.reduce((sum, item) => sum + item.price * item.qty, 0);
}

// Impure shell — small, obvious
function checkout(cart) {
  const total = calculateTotal(cart);
  document.querySelector("#total").textContent = total;
  localStorage.setItem("lastTotal", total);
}
```

Now `calculateTotal` can be unit-tested without a DOM or storage.

### React Connection

React's rule is exactly this split:

| Place                   | Must be                         |
| ----------------------- | ------------------------------- |
| Component render body   | Pure (compute JSX from props/state) |
| Reducers (`useReducer`, Redux) | Pure                     |
| `useEffect`, event handlers | Where side effects belong   |

In development, **`<StrictMode>` renders components twice** (and runs effects setup → cleanup → setup) specifically to expose impure render logic and missing cleanups.

## Interview Answer

> "A side effect is anything a function does other than return a value — mutating data, calling APIs, touching the DOM, logging, timers. Side effects are necessary, so the goal is to isolate them: keep the core logic pure and push effects to the edges. In React, rendering should be pure and effects belong in `useEffect` or event handlers."

## Key Takeaways

- Side effect = interaction with anything outside the function.
- Necessary, but should be isolated and explicit.
- Pure core + impure shell is the practical pattern.
- React: pure render, effects in `useEffect` / handlers; StrictMode double-invokes to catch mistakes.

---

# 5. Immutability (Supporting Concept)

Purity is impossible if you mutate shared data, so FP pairs with **immutability**: **don't change existing data — create new data**.

```js
const user = { name: "Asha", age: 25 };

// ❌ Mutation
user.age = 26;

// ✅ Immutable update
const updated = { ...user, age: 26 };
```

### Mutating vs Non-Mutating Array Methods

| Mutates ❌           | Returns new ✅                         |
| -------------------- | -------------------------------------- |
| `push`, `pop`        | `[...arr, x]`, `arr.slice(0, -1)`      |
| `shift`, `unshift`   | `arr.slice(1)`, `[x, ...arr]`          |
| `splice`             | `slice`, `filter`, `toSpliced` (ES2023) |
| `sort`               | `[...arr].sort()`, `toSorted` (ES2023) |
| `reverse`            | `[...arr].reverse()`, `toReversed` (ES2023) |
| `arr[i] = x`         | `arr.map(...)`, `arr.with(i, x)` (ES2023) |

### Spread is shallow

```js
const state = { user: { name: "Asha" } };
const copy = { ...state };

copy.user.name = "Ravi";
console.log(state.user.name); // "Ravi" ❗ nested object is shared
```

Nested updates must copy every level you change:

```js
const next = { ...state, user: { ...state.user, name: "Ravi" } };
```

> Full detail lives in `00-starter-roadmap.md` §5 (Shallow & Deep Copy). For now remember: **spread copies one level**.

`Object.freeze()` prevents changes, but is also **shallow** — nested objects can still be mutated.

---

# 6. Function Factory

## What is a Function Factory?

A **function factory** is a function that **creates and returns new, customized functions**. Each returned function remembers its own configuration through a **closure**.

```js
function createMultiplier(factor) {
  return (n) => n * factor;
}

const double = createMultiplier(2);
const triple = createMultiplier(3);

console.log(double(10)); // 20
console.log(triple(10)); // 30
```

```text
createMultiplier(2) ──▶ fn ──closure──▶ { factor: 2 }
createMultiplier(3) ──▶ fn ──closure──▶ { factor: 3 }
```

Each call creates a **separate** lexical environment, so `double` and `triple` don't interfere.

### Real-world: validators

```js
function minLength(min) {
  return (value) => value.length >= min || `Must be at least ${min} characters`;
}

function matches(regex, message) {
  return (value) => regex.test(value) || message;
}

const passwordRules = [minLength(8), matches(/\d/, "Must contain a number")];

function validate(value, rules) {
  return rules.map((rule) => rule(value)).filter((r) => r !== true);
}

console.log(validate("abc", passwordRules));
```

Output:

```text
[ 'Must be at least 8 characters', 'Must contain a number' ]
```

### Real-world: API clients / loggers

```js
function createLogger(prefix) {
  return (message) => console.log(`[${prefix}] ${message}`);
}

const authLog = createLogger("AUTH");
const cartLog = createLogger("CART");

authLog("User logged in"); // [AUTH] User logged in
cartLog("Item added");     // [CART] Item added
```

### Factory vs Class

| Function factory                         | Class                          |
| ---------------------------------------- | ------------------------------ |
| Private data via closure (truly private) | Private via `#field`           |
| No `this`, no `new`                      | Needs `new`, `this` can be lost |
| New function copies per instance         | Methods shared on prototype    |

## Interview Answer

> "A function factory is a higher-order function that returns new functions configured with the arguments it received. The returned functions keep access to that configuration through closures. It's used for things like validators, loggers, event handler creators and API clients."

## Key Takeaways

- Factory = function that returns customized functions.
- Configuration is remembered via closure.
- Each factory call → independent closure.
- Currying (Section 9) is a factory applied one argument at a time.

---

# 7. Function Composition

## What is Composition?

**Function composition** is combining small functions so that the **output of one becomes the input of the next**.

Math notation: `(f ∘ g)(x) = f(g(x))`

```js
const trim = (s) => s.trim();
const lower = (s) => s.toLowerCase();
const exclaim = (s) => s + "!";

// Manual composition — read inside-out
const result = exclaim(lower(trim("  HELLO  ")));
console.log(result); // "hello!"
```

Nested calls get hard to read as the chain grows. `compose` and `pipe` fix that.

### `compose` vs `pipe`

| Utility   | Direction                | Reads like      |
| --------- | ------------------------ | --------------- |
| `compose` | **Right → Left**         | Math `f(g(x))`  |
| `pipe`    | **Left → Right**         | A step-by-step pipeline (Unix `|`) |

```js
compose(exclaim, lower, trim)("  HELLO  "); // trim → lower → exclaim
pipe(trim, lower, exclaim)("  HELLO  ");    // trim → lower → exclaim
```

Same result, different reading order. Most people find **`pipe` easier to read**.

### Rule for composable functions

- Each function takes **one input** and returns **one output** (unary).
- Output type of step _n_ must fit input type of step _n+1_.
- Functions should be **pure** so the pipeline is predictable.

> This is why **currying** matters: it turns multi-argument functions into unary ones that fit into pipelines (Section 10).

---

# 8. Implement `compose()` and `pipe()` ⭐⭐⭐

### `pipe` — left to right

```js
const pipe =
  (...fns) =>
  (x) =>
    fns.reduce((acc, fn) => fn(acc), x);
```

### `compose` — right to left

```js
const compose =
  (...fns) =>
  (x) =>
    fns.reduceRight((acc, fn) => fn(acc), x);
```

The only difference: `reduce` vs `reduceRight`.

### Dry run

```js
const add1 = (x) => x + 1;
const double = (x) => x * 2;

pipe(add1, double)(5);
```

```text
reduce starts with acc = 5
  add1(5)    → 6
  double(6)  → 12
result: 12
```

```js
compose(add1, double)(5);
```

```text
reduceRight starts with acc = 5
  double(5)  → 10
  add1(10)   → 11
result: 11
```

### Edge case: no functions

```js
pipe()(7);    // 7
compose()(7); // 7
```

With an empty list, `reduce` just returns the initial value → behaves like the **identity function**. Interviewers like this edge case.

### Follow-up: first function takes multiple arguments

```js
const pipeN =
  (...fns) =>
  (...args) => {
    if (fns.length === 0) return args[0];
    const [first, ...rest] = fns;
    return rest.reduce((acc, fn) => fn(acc), first(...args));
  };

const sum = (a, b) => a + b;
pipeN(sum, double)(2, 3); // double(sum(2, 3)) → 10
```

### Follow-up: async pipe

Each step may return a Promise (e.g. fetch → parse → transform).

```js
const pipeAsync =
  (...fns) =>
  (x) =>
    fns.reduce((promise, fn) => promise.then(fn), Promise.resolve(x));

pipeAsync(
  async (x) => x + 1,
  (x) => x * 2,
)(5).then(console.log); // 12
```

`.then` accepts both sync and async functions, so steps can be mixed.

### Practical: data pipeline

```js
const users = [
  { name: "asha", age: 17, active: true },
  { name: "ravi", age: 25, active: true },
  { name: "neha", age: 31, active: false },
  { name: "karan", age: 22, active: true },
];

const filterBy = curry((pred, arr) => arr.filter(pred));
const mapBy = curry((fn, arr) => arr.map(fn));
const sortBy = curry((key, arr) =>
  [...arr].sort((a, b) => (a[key] > b[key] ? 1 : a[key] < b[key] ? -1 : 0)),
);

const getActiveAdultNames = pipe(
  filterBy((u) => u.active),
  filterBy((u) => u.age >= 18),
  sortBy("name"),
  mapBy((u) => u.name.toUpperCase()),
);

console.log(getActiveAdultNames(users)); // [ 'KARAN', 'RAVI' ]
console.log(users[0].name);              // 'asha' — original untouched
```

(`curry` is implemented in Section 10.) Notice `sortBy` copies first with `[...arr]` because `sort` mutates.

## Interview Answer

> "Function composition means combining functions so the output of one becomes the input of the next. `compose` applies functions right to left like math notation, `pipe` applies them left to right like a pipeline. Both can be implemented with `reduce` — `pipe` with `reduce`, `compose` with `reduceRight` — starting from the initial value."

## Key Takeaways

- Composition = chaining outputs into inputs.
- `compose` = right→left (`reduceRight`), `pipe` = left→right (`reduce`).
- Empty `pipe()` / `compose()` → identity.
- Works best with pure, unary functions → currying helps.
- Async version: chain with `.then` starting from `Promise.resolve(x)`.

---

# 9. Currying

## What is Currying?

**Currying** transforms a function that takes **multiple arguments** into a **sequence of functions that each take one argument**.

```text
f(a, b, c)   ──curry──▶   f(a)(b)(c)
```

```js
// Normal
function add(a, b, c) {
  return a + b + c;
}
add(1, 2, 3); // 6

// Curried manually
function addCurried(a) {
  return function (b) {
    return function (c) {
      return a + b + c;
    };
  };
}
addCurried(1)(2)(3); // 6

// Arrow-function version
const addArrow = (a) => (b) => (c) => a + b + c;
```

### How it works internally

Each call returns a new function that **closes over** the arguments received so far.

```text
addCurried(1)       → fn  closure { a: 1 }
addCurried(1)(2)    → fn  closure { b: 2 } → outer { a: 1 }
addCurried(1)(2)(3) → a + b + c = 6
```

Currying = **closures + function factories**, applied one argument at a time.

### Why Currying is Useful

**1. Reusable specialized functions**

```js
const discount = (percent) => (price) => price - price * (percent / 100);

const tenOff = discount(10);
const blackFriday = discount(50);

tenOff(200);      // 180
blackFriday(200); // 100
```

**2. Makes functions fit into `map` / `pipe`** (they expect one argument)

```js
const multiply = (a) => (b) => a * b;
[1, 2, 3].map(multiply(10)); // [10, 20, 30]
```

**3. Configure once, use many times**

```js
const request = (baseUrl) => (endpoint) => fetch(`${baseUrl}${endpoint}`);

const api = request("https://api.example.com");
api("/users");
api("/orders");
```

**4. React event handlers**

```js
const handleChange = (field) => (event) => {
  setForm((prev) => ({ ...prev, [field]: event.target.value }));
};

<input onChange={handleChange("email")} />
<input onChange={handleChange("password")} />
```

> Note: `handleChange("email")` creates a **new function each render**. That's fine for normal inputs, but if passed to a `React.memo` child it will break memoization.

---

# 10. Implement `curry()` ⭐⭐⭐

Goal: a generic `curry(fn)` that supports **all** of these:

```js
const curriedAdd = curry((a, b, c) => a + b + c);

curriedAdd(1)(2)(3); // 6
curriedAdd(1, 2)(3); // 6
curriedAdd(1)(2, 3); // 6
curriedAdd(1, 2, 3); // 6
```

(Strictly, accepting several args at once is "auto-currying", which is what Lodash's `curry` and interviewers expect.)

### Implementation

```js
function curry(fn) {
  return function curried(...args) {
    // Enough arguments collected → call the original
    if (args.length >= fn.length) {
      return fn.apply(this, args);
    }

    // Not enough → return a function that collects more
    return function (...nextArgs) {
      return curried.apply(this, [...args, ...nextArgs]);
    };
  };
}
```

### Key ideas

- **`fn.length`** = number of declared parameters (the "arity"). That's how we know when to stop.
- **`args` accumulate** via closure across calls.
- **`apply(this, ...)`** keeps `this` working if the curried function is used as a method.
- **`[...args, ...nextArgs]`** creates a new array each time, so partial results are **reusable**:

```js
const add5 = curriedAdd(5);
add5(1)(1); // 7
add5(2)(2); // 9  ← not polluted by the previous call
```

### Dry run: `curriedAdd(1)(2, 3)`

```text
curried(1)         args=[1]        1 < 3 → return collector
collector(2, 3)    curried(1,2,3)  3 >= 3 → fn(1,2,3) = 6
```

### ⚠️ `fn.length` Gotchas (great follow-up question)

`fn.length` counts only parameters **before** the first default value or rest parameter.

```js
((a, b, c) => {}).length;      // 3
((a, b = 10, c) => {}).length; // 1  ← stops at the default
((...args) => {}).length;      // 0
```

So:

```js
const add = (a, b = 10, c) => a + b + c;
curry(add)(1); // NaN — called immediately with a=1, b=10, c=undefined
```

**Fix:** let the caller pass the arity explicitly.

```js
function curry(fn, arity = fn.length) {
  return function curried(...args) {
    if (args.length >= arity) return fn.apply(this, args);
    return (...next) => curried.apply(this, [...args, ...next]);
  };
}
```

### Bonus: Curry with Placeholders (advanced follow-up)

Lets you skip an argument and fill it in later: `curried(_, 2)(1)` → `fn(1, 2)`.

```js
const _ = Symbol("placeholder");

function curryWithPlaceholder(fn) {
  return function curried(...args) {
    const complete =
      args.length >= fn.length && args.slice(0, fn.length).every((a) => a !== _);

    if (complete) return fn.apply(this, args);

    return (...next) => {
      // Fill placeholders from left to right with the new args
      const merged = args.map((a) => (a === _ && next.length ? next.shift() : a));
      return curried.apply(this, [...merged, ...next]);
    };
  };
}

const f = curryWithPlaceholder((a, b, c) => [a, b, c]);
f(_, 2)(1)(3);    // [1, 2, 3]
f(_, _, 3)(1)(2); // [1, 2, 3]
f(1)(_, 3)(2);    // [1, 2, 3]
```

---

## 10.1 Infinite Currying — `sum(1)(2)(3)()` ⭐⭐⭐

Very common interview question: support **any number** of calls.

### Version A — empty call ends the chain

```js
function sum(a) {
  return function next(b) {
    if (b === undefined) return a; // sum(...)() → final result
    return sum(a + b);
  };
}

sum(1)(2)(3)();  // 6
sum(1)(2)(0)();  // 3  ← 0 works because we check undefined, not falsy
sum(5)();        // 5
```

> ❗ Common bug: `if (!b) return a;` — breaks when someone passes `0`.

### Version B — no ending call, using `valueOf`

```js
function sum(...nums) {
  const total = nums.reduce((a, b) => a + b, 0);
  const next = (...more) => sum(total, ...more);
  next.valueOf = () => total; // used when JS converts to a primitive
  return next;
}

+sum(1)(2)(3);       // 6
sum(1, 2)(3) == 6;   // true (loose equality triggers valueOf)
```

This relies on **type coercion** (you covered `valueOf` / `ToPrimitive` in fundamentals). Mention that it's clever but less explicit than Version A.

## Interview Answer

> "Currying transforms a function with multiple arguments into a chain of functions that each take one argument, like `f(a)(b)(c)`. It works using closures — each returned function remembers the arguments collected so far. A generic `curry` compares the collected arguments with `fn.length` and calls the original once there are enough. It's useful for creating reusable specialized functions and for making functions fit into `map` or `pipe`."

## Key Takeaways

- Currying: `f(a, b, c)` → `f(a)(b)(c)`.
- Implemented with closures that accumulate arguments.
- Generic `curry` uses `fn.length` to know when to call.
- `fn.length` ignores params after a default and rest params → accept an explicit arity.
- Infinite currying: end with `()` (check `undefined`, not falsy) or use `valueOf`.

---

# 11. Partial Application

## What is Partial Application?

**Partial application** = fixing **some** arguments of a function now and returning a function that takes **the rest** later.

```js
function greet(greeting, punctuation, name) {
  return `${greeting}, ${name}${punctuation}`;
}

const sayHello = partial(greet, "Hello", "!");
sayHello("Asha"); // "Hello, Asha!"
```

### Implement `partial()`

```js
const partial =
  (fn, ...presetArgs) =>
  (...laterArgs) =>
    fn(...presetArgs, ...laterArgs);
```

### `bind` is built-in partial application

```js
function multiply(a, b) {
  return a * b;
}

const double = multiply.bind(null, 2); // this = null, a = 2
double(7); // 14
```

### Currying vs Partial Application ⭐⭐⭐ (very common question)

| Currying                                     | Partial application                     |
| -------------------------------------------- | --------------------------------------- |
| Converts `f(a, b, c)` → `f(a)(b)(c)`         | Fixes some args: `f(a, b, c)` → `g(c)`  |
| Returns a chain of (ideally) **unary** functions | Returns **one** function for the remaining args |
| Calls continue until all args are collected  | Usually just one extra call             |
| A **transformation of the function**         | A **use** of the function with some args filled in |

```js
const add3 = (a, b, c) => a + b + c;

// Currying
curry(add3)(1)(2)(3); // 6

// Partial application
partial(add3, 1, 2)(3); // 6
```

> Simple way to say it: **a curried function lets you partially apply it one argument at a time. Partial application can fix any number of arguments in one go.**

## Key Takeaways

- Partial application = pre-fill some arguments, return function for the rest.
- `bind(thisArg, ...args)` does partial application natively.
- Currying ≠ partial application, but currying makes partial application easy.

---

# 12. `once()` ⭐⭐⭐

## What is `once`?

`once(fn)` returns a function that runs `fn` **only the first time** it's called. Later calls return the **first result**.

Use cases:

- Initialize an SDK / analytics only once
- Prevent double form submission
- Attach a one-time setup handler
- Lazy singletons

### Implementation

```js
function once(fn) {
  let called = false;
  let result;

  return function (...args) {
    if (!called) {
      called = true;
      result = fn.apply(this, args);
      fn = null; // release reference → allow GC of fn and its closure
    }
    return result;
  };
}
```

```js
const init = once((x) => x * 2);

init(5);  // 10
init(10); // 10 — fn not called again, cached result returned
```

### Interview details to mention

| Detail                       | Why                                                      |
| ---------------------------- | -------------------------------------------------------- |
| `called` flag, not `result`  | `fn` may legitimately return `undefined` / `0` / `false` |
| `called = true` **before** calling `fn` | Prevents re-entry if `fn` calls itself recursively |
| `fn.apply(this, args)`       | Works correctly as an object method                      |
| `fn = null`                  | Memory: lets GC collect `fn` and what it closes over     |
| Throw behaviour              | If `fn` throws, `called` is already true → later calls return `undefined`. Mention it and say you could reset `called` in a `catch` if retries should be allowed |

### Works with `this`

```js
const counter = {
  count: 0,
  inc: once(function () {
    return ++this.count;
  }),
};

counter.inc(); // 1
counter.inc(); // 1
counter.count; // 1
```

### Built-in alternative for events

```js
button.addEventListener("click", handler, { once: true });
```

## Key Takeaways

- `once` = HOF + closure holding a flag and the result.
- Use a boolean flag, not "is result undefined".
- Preserve `this` and arguments with `apply`.
- Release `fn` after the call for memory.

---

# 13. Point-Free Style (Bonus Vocabulary)

**Point-free** means defining functions without mentioning their arguments.

```js
// With argument ("pointed")
const getNames = (users) => users.map((u) => u.name);

// Point-free
const prop = (key) => (obj) => obj[key];
const getNames2 = mapBy(prop("name"));
```

Nice for short pipelines, but harder to read and debug if overused. Say this in interviews — it shows judgment.

### ⚠️ The `map(parseInt)` trap

Passing a function directly (point-free) to `map` can bite you because `map` passes **three** arguments: `(value, index, array)`.

```js
["1", "2", "3"].map(parseInt);
```

Output:

```text
[1, NaN, NaN]
```

Because it actually calls:

```text
parseInt("1", 0) → 1    (radix 0 → treated as 10)
parseInt("2", 1) → NaN  (radix 1 is invalid)
parseInt("3", 2) → NaN  ("3" isn't a binary digit)
```

**Fix:** `["1", "2", "3"].map((s) => parseInt(s, 10))` or `.map(Number)`.

---

# 14. Functional Programming in React ⚛️

React is heavily inspired by FP. Interviewers love asking you to connect the two.

### 1. Components should be pure functions of props/state

```jsx
// ✅ Pure: same props → same JSX
function Price({ amount, currency }) {
  return <span>{new Intl.NumberFormat("en-IN", { style: "currency", currency }).format(amount)}</span>;
}
```

```jsx
// ❌ Impure render: mutates outside variable
let renderCount = 0;
function Bad() {
  renderCount++; // side effect during render
  return <p>{renderCount}</p>;
}
```

`UI = f(state)` — this is React's core idea.

### 2. Reducers must be pure

```js
function cartReducer(state, action) {
  switch (action.type) {
    case "add":
      return { ...state, items: [...state.items, action.item] }; // ✅ new object
    case "remove":
      return { ...state, items: state.items.filter((i) => i.id !== action.id) };
    default:
      return state;
  }
}
```

❌ Never `state.items.push(...)` in a reducer — React compares references, so a mutated object looks "unchanged" and **won't re-render**.

### 3. Immutability → change detection

React (and `React.memo`, `useMemo`, `useEffect` deps) uses `Object.is` reference comparison. Immutable updates create new references, which is how React knows something changed.

### 4. Higher-Order Components = HOF for components

```jsx
function withLoading(Component) {
  return function WithLoading({ isLoading, ...props }) {
    if (isLoading) return <Spinner />;
    return <Component {...props} />;
  };
}

const UserListWithLoading = withLoading(UserList);
```

Composing HOCs is literally `compose`:

```js
const Enhanced = compose(withAuth, withTheme, withLoading)(Dashboard);
// = withAuth(withTheme(withLoading(Dashboard)))
```

> Modern React prefers **custom hooks** over HOCs, but HOCs still appear in older codebases and interviews.

### 5. Custom hooks = composition of behaviour

```jsx
function useSearch(query) {
  const debounced = useDebounce(query, 300); // compose hooks
  return useFetch(`/api/search?q=${debounced}`);
}
```

### 6. Array HOFs render lists

```jsx
{users
  .filter((u) => u.active)
  .map((u) => <UserRow key={u.id} user={u} />)}
```

### 7. Functional state updates

```js
setCount((prev) => prev + 1); // pass a pure function of previous state
```

Avoids stale closures (see `02a-closures.md` §23–25).

## Interview Answer — "How does React use functional programming?"

> "React treats components as pure functions of props and state — `UI = f(state)`. Rendering must be pure; side effects go in `useEffect` or event handlers. State is updated immutably so React can detect changes by reference. Reducers are pure functions. HOCs are higher-order functions for components, custom hooks compose behaviour, and lists are rendered with `map` and `filter`."

---

# 15. Output-Based Questions 🧪

Try each **before** reading the answer.

### Q1

```js
const nums = [1, 2, 3];
const doubled = nums.map((n) => n * 2);
nums.push(4);
console.log(doubled, nums);
```

<details><summary>Answer</summary>

```text
[2, 4, 6] [1, 2, 3, 4]
```

`map` returns a **new** array; later mutation of `nums` doesn't affect it.

</details>

### Q2

```js
const a = [3, 1, 2];
const b = a.sort();
console.log(a === b, a);
```

<details><summary>Answer</summary>

```text
true [1, 2, 3]
```

`sort` mutates **in place** and returns the **same** array. Impure! Use `[...a].sort()` or `a.toSorted()`.

</details>

### Q3

```js
console.log([10, 1, 2].sort());
```

<details><summary>Answer</summary>

```text
[1, 10, 2]
```

Default sort converts to **strings** and compares lexicographically. Use `.sort((x, y) => x - y)` → `[1, 2, 10]`.

</details>

### Q4

```js
console.log(["1", "2", "3"].map(parseInt));
```

<details><summary>Answer</summary>

```text
[1, NaN, NaN]
```

`map` passes `(value, index)`; index becomes `parseInt`'s radix. See Section 13.

</details>

### Q5

```js
const add1 = (x) => x + 1;
const double = (x) => x * 2;

console.log(compose(add1, double)(5));
console.log(pipe(add1, double)(5));
```

<details><summary>Answer</summary>

```text
11
12
```

`compose` → `add1(double(5))`. `pipe` → `double(add1(5))`.

</details>

### Q6

```js
const init = once((x) => x * 2);
console.log(init(5), init(10));
```

<details><summary>Answer</summary>

```text
10 10
```

Second call returns the cached first result.

</details>

### Q7

```js
const add = (a, b = 10, c) => a + b + c;
console.log(add.length);
console.log(curry(add)(1));
```

<details><summary>Answer</summary>

```text
1
NaN
```

`length` stops at the first default param → 1. `curry` sees 1 arg ≥ 1 and calls immediately: `1 + 10 + undefined = NaN`.

</details>

### Q8

```js
function makeAdders() {
  const fns = [];
  for (var i = 0; i < 3; i++) {
    fns.push((x) => x + i);
  }
  return fns;
}

const [a0, , a2] = makeAdders();
console.log(a0(10), a2(10));
```

<details><summary>Answer</summary>

```text
13 13
```

Function factory + `var` trap: all closures share **one** `i`, which is `3` after the loop. With `let`, the output is `10 12`.

</details>

### Q9

```js
console.log([].reduce((a, b) => a + b));
```

<details><summary>Answer</summary>

```text
TypeError: Reduce of empty array with no initial value
```

Always pass an initial value: `.reduce((a, b) => a + b, 0)`.

</details>

### Q10

```js
const state = { user: { name: "Asha" } };
const copy = { ...state };
copy.user.name = "Ravi";
console.log(state.user.name, state === copy, state.user === copy.user);
```

<details><summary>Answer</summary>

```text
Ravi false true
```

Spread is **shallow**: new outer object, but `user` is the same reference.

</details>

### Q11

```js
function sum(a) {
  return function (b) {
    if (!b) return a;
    return sum(a + b);
  };
}
console.log(sum(1)(2)(0)(3)());
```

<details><summary>Answer</summary>

```text
TypeError: sum(...)(...)(...) is not a function
```

Bug: `!b` treats `0` as "end of chain". So `sum(1)(2)(0)` returns the **number** `3`, and then calling `(3)` on a number throws. This is exactly why you check `b === undefined` instead of falsy.

</details>

---

# 16. Implement From Scratch — Practice Set 🛠️

Do these **without looking**, then compare with the sections above.

| #   | Problem                                                     | Priority | Reference  |
| --- | ----------------------------------------------------------- | -------- | ---------- |
| 1   | `pipe(...fns)`                                              | ⭐⭐⭐   | §8         |
| 2   | `compose(...fns)`                                           | ⭐⭐⭐   | §8         |
| 3   | `curry(fn)` supporting `f(1)(2)(3)` and `f(1, 2)(3)`        | ⭐⭐⭐   | §10        |
| 4   | `sum(1)(2)(3)()` — infinite currying                        | ⭐⭐⭐   | §10.1      |
| 5   | `once(fn)` preserving `this`                                | ⭐⭐⭐   | §12        |
| 6   | `partial(fn, ...args)`                                      | ⭐       | §11        |
| 7   | `pipeAsync(...fns)`                                         | ⭐       | §8         |
| 8   | `curry` with explicit arity                                 | ⭐       | §10        |
| 9   | Factory `createCounter()` with `inc`, `dec`, `reset`        | ⭐       | §6         |
| 10  | Rewrite an impure function (mutation + global) to be pure   | ⭐       | §3         |
| 11  | Build a pipeline: active users → adults → sorted → names    | ⭐       | §8         |
| 12  | `curry` with placeholders                                   | Bonus    | §10        |

### Test cases to verify your answers

```js
// pipe / compose
console.assert(pipe((x) => x + 1, (x) => x * 2)(5) === 12);
console.assert(compose((x) => x + 1, (x) => x * 2)(5) === 11);
console.assert(pipe()(7) === 7);

// curry
const c = curry((a, b, c) => a + b + c);
console.assert(c(1)(2)(3) === 6 && c(1, 2)(3) === 6 && c(1)(2, 3) === 6 && c(1, 2, 3) === 6);
const add5 = c(5);
console.assert(add5(1)(1) === 7 && add5(2)(2) === 9);

// infinite sum
console.assert(sum(1)(2)(3)() === 6 && sum(1)(2)(0)() === 3 && sum(5)() === 5);

// once
let runs = 0;
const o = once(() => ++runs);
o(); o(); o();
console.assert(runs === 1);

console.log("All tests ran — no assertion messages above means you passed ✅");
```

---

# 17. Common Mistakes ❌

| Mistake                                        | Correct thinking                                 |
| ---------------------------------------------- | ------------------------------------------------ |
| "Pure means no `console.log` only"             | Pure = deterministic **and** no side effects of any kind |
| Using `sort`/`reverse`/`splice` in "pure" code | They mutate → copy first or use `toSorted` etc.  |
| Thinking spread = deep copy                    | Spread is shallow                                |
| Mixing up `compose` and `pipe` direction       | compose = right→left, pipe = left→right          |
| Saying currying = partial application          | Related but different (§11 table)                |
| `curry` relying on `fn.length` with defaults/rest | Accept explicit arity                         |
| `if (!b)` in infinite sum                      | Check `b === undefined`                          |
| `once` checking `result === undefined`         | Use a separate boolean flag                      |
| Passing `parseInt` directly to `map`           | Wrap it: `(s) => parseInt(s, 10)`                |
| Side effects inside React render or reducer    | Move to `useEffect` / handlers                   |
| "FP means never mutate anything ever"          | Local mutation inside a function that's invisible outside is fine; the function stays pure |

---

# 18. Interview Follow-Up Questions

Be ready to answer these out loud (30–60 seconds each):

1. What is a higher-order function? Give 3 built-in examples.
2. What makes a function pure? Is `console.log` inside a function a side effect?
3. Is `Array.prototype.sort` pure? How do you sort without mutating?
4. Why should React components be pure? What does StrictMode do about it?
5. What's the difference between `compose` and `pipe`? Implement both.
6. What is currying? Implement a generic `curry`.
7. What's the difference between currying and partial application?
8. Why is `fn.length` unreliable for currying?
9. Implement `sum(1)(2)(3)()`. How would you make `sum(1)(2)(3) == 6` work without `()`?
10. Implement `once`. Why use a flag instead of checking the result?
11. How does `bind` relate to partial application?
12. What is a function factory? How is it different from a class?
13. Can a function that mutates a local array still be pure?
    → **Yes**, if the array is created inside the function and never escapes until returned.
14. What is referential transparency?
15. How do HOCs relate to higher-order functions?
16. Why does `['1','2','3'].map(parseInt)` return `[1, NaN, NaN]`?

---

# 19. Final Cheat Sheet 📋

```text
HOF            takes fn and/or returns fn            map, filter, debounce, bind
Pure fn        same input → same output, no effects  add(a, b)
Side effect    anything outside return value         fetch, DOM, log, mutate, timers
Immutability   create new data, don't change old     {...obj}, [...arr], toSorted
Factory        fn that returns configured fns        createMultiplier(2)
Composition    output of one → input of next         f(g(x))
compose        right → left                          reduceRight
pipe           left → right                          reduce
Currying       f(a,b,c) → f(a)(b)(c)                 closures + fn.length
Partial        pre-fill some args                    partial(fn, a), fn.bind(null, a)
once           run only first time, cache result     flag + result in closure
```

```js
const pipe    = (...fns) => (x) => fns.reduce((acc, fn) => fn(acc), x);
const compose = (...fns) => (x) => fns.reduceRight((acc, fn) => fn(acc), x);
const partial = (fn, ...a) => (...b) => fn(...a, ...b);

function curry(fn, arity = fn.length) {
  return function curried(...args) {
    if (args.length >= arity) return fn.apply(this, args);
    return (...next) => curried.apply(this, [...args, ...next]);
  };
}

function once(fn) {
  let called = false, result;
  return function (...args) {
    if (!called) { called = true; result = fn.apply(this, args); fn = null; }
    return result;
  };
}

const sum = (a) => (b) => (b === undefined ? a : sum(a + b));
```

---

# 20. Readiness Checklist ✅

Mark Functional Programming **completed** in `00-starter-roadmap.md` only when you can:

- [ ] Explain HOF, pure function, side effect in 30 seconds each
- [ ] Explain why purity matters for testing, memoization and React
- [ ] Write `pipe`, `compose`, `curry`, `once`, `partial` from memory
- [ ] Write infinite `sum(1)(2)(3)()` and explain the `0` edge case
- [ ] Explain currying vs partial application with an example
- [ ] Explain `fn.length` gotchas
- [ ] Solve all 11 output questions without looking
- [ ] Connect FP to React: pure render, pure reducers, immutable updates, HOCs, hooks

## Final Mental Model

```text
Small pure functions
        +
Immutable data
        +
Side effects at the edges
        +
Combine with HOFs, currying and composition
        =
Predictable, testable, reusable code (and how React thinks)
```
