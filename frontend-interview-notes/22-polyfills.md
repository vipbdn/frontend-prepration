# 22 — Polyfills 🔥🔥🔥

> **Interview + Understanding Guide** — `00-starter-roadmap.md` Section 22
>
> Goal: Write spec-aware polyfills for array methods, `call/apply/bind`, and Promise statics — and explain the edge cases interviewers probe (holes, `thisArg`, `reduce` without initial value, `new` with `bind`).

---

# 1. What is a Polyfill?

A **polyfill** is code that implements a feature on environments that **don't support it natively**, using only older features.

```js
if (!Array.prototype.flat) {
  Array.prototype.flat = function (/* ... */) { /* ... */ };
}
```

| Polyfill | Transpiler (Babel) |
| --- | --- |
| Adds missing **APIs** at runtime (`Array.prototype.includes`, `Promise`) | Rewrites new **syntax** at build time (`=>`, `class`, `?.`) |
| core-js | @babel/preset-env |

In interviews, "write a polyfill for `map`" means: **implement it yourself** — usually as `Array.prototype.myMap` so you don't overwrite the real one.

> ⚠️ In real apps, don't modify built-in prototypes casually ("monkey patching") — it can clash with future standards (the famous `Array.prototype.flatten` → `flat` rename happened because of a library doing this).

### General rules for array polyfills

1. `this` is the array → use `const arr = this` (and **regular functions**, not arrows!).
2. Validate the callback: `if (typeof cb !== "function") throw new TypeError(...)`.
3. Support `thisArg`: `cb.call(thisArg, element, index, array)`.
4. Pass **3 arguments**: `(element, index, array)`.
5. **Skip holes** in sparse arrays: `if (i in arr)` (or `Object.hasOwn`/`hasOwnProperty`).
6. Capture `length` **once** at the start (spec behaviour).
7. Don't mutate the original array (unless the method does).

---

# 2. Array Polyfills ⭐⭐⭐

## 2.1 `map`

```js
Array.prototype.myMap = function (callback, thisArg) {
  if (typeof callback !== "function") {
    throw new TypeError(`${callback} is not a function`);
  }

  const arr = this;
  const len = arr.length;
  const result = new Array(len); // same length; holes stay holes

  for (let i = 0; i < len; i++) {
    if (i in arr) {
      result[i] = callback.call(thisArg, arr[i], i, arr);
    }
  }
  return result;
};

[1, 2, 3].myMap((x) => x * 2); // [2, 4, 6]
[1, , 3].myMap((x) => x * 2);  // [2, <empty>, 6]
```

## 2.2 `filter`

```js
Array.prototype.myFilter = function (callback, thisArg) {
  if (typeof callback !== "function") {
    throw new TypeError(`${callback} is not a function`);
  }

  const arr = this;
  const len = arr.length;
  const result = [];

  for (let i = 0; i < len; i++) {
    if (i in arr && callback.call(thisArg, arr[i], i, arr)) {
      result.push(arr[i]); // truthy return → keep
    }
  }
  return result;
};

[1, 2, 3, 4].myFilter((x) => x % 2 === 0); // [2, 4]
```

## 2.3 `reduce` ⭐⭐⭐ (most edge cases)

```js
Array.prototype.myReduce = function (callback, initialValue) {
  if (typeof callback !== "function") {
    throw new TypeError(`${callback} is not a function`);
  }

  const arr = this;
  const len = arr.length;
  let i = 0;
  let acc;

  if (arguments.length >= 2) {
    // initialValue explicitly provided — even if it's undefined!
    acc = initialValue;
  } else {
    // No initial value: use the first existing element
    while (i < len && !(i in arr)) i++; // skip leading holes
    if (i >= len) {
      throw new TypeError("Reduce of empty array with no initial value");
    }
    acc = arr[i++];
  }

  for (; i < len; i++) {
    if (i in arr) {
      acc = callback(acc, arr[i], i, arr); // 4 args: acc, current, index, array
    }
  }
  return acc;
};

[1, 2, 3].myReduce((a, b) => a + b);     // 6
[1, 2, 3].myReduce((a, b) => a + b, 10); // 16
[].myReduce((a, b) => a + b, 0);         // 0
[].myReduce((a, b) => a + b);            // TypeError
```

### Why `arguments.length` and not `initialValue === undefined`?

```js
[1, 2].reduce((acc, x) => acc, undefined); // undefined — initial value WAS provided
```

Checking `initialValue === undefined` would wrongly treat this as "no initial value" and return `1`.

> `reduce` has **no `thisArg`** parameter. `reduceRight` is the same but iterates from the end.

## 2.4 `forEach`

```js
Array.prototype.myForEach = function (callback, thisArg) {
  if (typeof callback !== "function") {
    throw new TypeError(`${callback} is not a function`);
  }
  const arr = this;
  const len = arr.length;
  for (let i = 0; i < len; i++) {
    if (i in arr) callback.call(thisArg, arr[i], i, arr);
  }
  // returns undefined
};
```

## 2.5 `find` / `findIndex`

```js
Array.prototype.myFind = function (callback, thisArg) {
  const arr = this;
  for (let i = 0; i < arr.length; i++) {
    // ❗ find does NOT skip holes — visits every index (holes = undefined)
    if (callback.call(thisArg, arr[i], i, arr)) return arr[i];
  }
  return undefined;
};

Array.prototype.myFindIndex = function (callback, thisArg) {
  const arr = this;
  for (let i = 0; i < arr.length; i++) {
    if (callback.call(thisArg, arr[i], i, arr)) return i;
  }
  return -1;
};
```

## 2.6 `some` / `every`

```js
Array.prototype.mySome = function (callback, thisArg) {
  const arr = this;
  for (let i = 0; i < arr.length; i++) {
    if (i in arr && callback.call(thisArg, arr[i], i, arr)) return true; // short-circuit
  }
  return false;
};

Array.prototype.myEvery = function (callback, thisArg) {
  const arr = this;
  for (let i = 0; i < arr.length; i++) {
    if (i in arr && !callback.call(thisArg, arr[i], i, arr)) return false; // short-circuit
  }
  return true; // vacuously true for []
};
```

## 2.7 Bonus: `flat`, `includes`

```js
Array.prototype.myFlat = function (depth = 1) {
  const result = [];
  const flatten = (arr, d) => {
    for (let i = 0; i < arr.length; i++) {
      if (!(i in arr)) continue; // flat removes holes
      if (Array.isArray(arr[i]) && d > 0) flatten(arr[i], d - 1);
      else result.push(arr[i]);
    }
  };
  flatten(this, depth);
  return result;
};

Array.prototype.myIncludes = function (search, fromIndex = 0) {
  const arr = this;
  const len = arr.length;
  let i = fromIndex >= 0 ? fromIndex : Math.max(len + fromIndex, 0);
  for (; i < len; i++) {
    // SameValueZero: like === but NaN equals NaN
    if (arr[i] === search || (Number.isNaN(arr[i]) && Number.isNaN(search))) return true;
  }
  return false;
};
```

### Quick reference — hole handling

| Skips holes | Visits holes as `undefined` |
| --- | --- |
| `map`, `filter`, `forEach`, `reduce`, `some`, `every`, `flat` | `find`, `findIndex`, `includes`, `for...of`, spread |

---

# 3. Function Polyfills ⭐⭐⭐

Full explanation of `this` rules in `11-this-and-function-binding.md`.

## 3.1 `call`

```js
Function.prototype.myCall = function (context, ...args) {
  if (typeof this !== "function") throw new TypeError("Not a function");

  context = context ?? globalThis;   // null/undefined → global (sloppy-mode behaviour)
  context = Object(context);          // box primitives so we can attach a property

  const fnKey = Symbol("fn");         // unique key → no collisions with existing props
  context[fnKey] = this;              // `this` = the function being called
  const result = context[fnKey](...args); // called as a method → this = context
  delete context[fnKey];
  return result;
};
```

## 3.2 `apply`

```js
Function.prototype.myApply = function (context, argsArray) {
  if (typeof this !== "function") throw new TypeError("Not a function");
  if (argsArray != null && typeof argsArray !== "object") {
    throw new TypeError("CreateListFromArrayLike called on non-object");
  }

  context = Object(context ?? globalThis);
  const fnKey = Symbol("fn");
  context[fnKey] = this;
  const result = context[fnKey](...Array.from(argsArray ?? [])); // supports array-likes
  delete context[fnKey];
  return result;
};
```

## 3.3 `bind` (with `new` support)

```js
Function.prototype.myBind = function (context, ...boundArgs) {
  if (typeof this !== "function") throw new TypeError("Bind must be called on a function");
  const targetFn = this;

  function boundFn(...callArgs) {
    // If called with `new`, `this` is an instance of boundFn → use it, ignore context
    const isConstructorCall = this instanceof boundFn;
    return targetFn.apply(isConstructorCall ? this : context, [...boundArgs, ...callArgs]);
  }

  // Keep the prototype chain for `new boundFn()`
  if (targetFn.prototype) {
    boundFn.prototype = Object.create(targetFn.prototype);
  }
  return boundFn;
};
```

### Tests

```js
function greet(greeting, punct) {
  return `${greeting}, ${this.name}${punct}`;
}
const asha = { name: "Asha" };

greet.myCall(asha, "Hi", "!");        // "Hi, Asha!"
greet.myApply(asha, ["Hello", "."]);  // "Hello, Asha."
greet.myBind(asha, "Hey")("?");       // "Hey, Asha?"

function Person(name, age) { this.name = name; this.age = age; }
const BoundPerson = Person.myBind({ ignored: true }, "Ravi");
const p = new BoundPerson(30);
p.name, p.age, p instanceof Person;   // "Ravi", 30, true
```

---

# 4. Promise Polyfills ⭐⭐⭐

Full explanations, edge cases and a custom `MyPromise` class in `06-promises.md` §7.

```js
Promise.myAll = function (iterable) {
  return new Promise((resolve, reject) => {
    const items = Array.from(iterable);
    const results = new Array(items.length);
    let remaining = items.length;
    if (remaining === 0) return resolve(results);

    items.forEach((item, index) => {
      Promise.resolve(item).then((value) => {
        results[index] = value;                 // preserve order
        if (--remaining === 0) resolve(results); // count — don't use results.length
      }, reject);                                // fail fast
    });
  });
};

Promise.myRace = function (iterable) {
  return new Promise((resolve, reject) => {
    for (const item of iterable) Promise.resolve(item).then(resolve, reject);
  });
};

Promise.myAllSettled = function (iterable) {
  return Promise.myAll(
    Array.from(iterable, (item) =>
      Promise.resolve(item).then(
        (value) => ({ status: "fulfilled", value }),
        (reason) => ({ status: "rejected", reason }),
      ),
    ),
  );
};

Promise.myAny = function (iterable) {
  return new Promise((resolve, reject) => {
    const items = Array.from(iterable);
    const errors = new Array(items.length);
    let remaining = items.length;
    if (remaining === 0) return reject(new AggregateError([], "All promises were rejected"));

    items.forEach((item, index) => {
      Promise.resolve(item).then(resolve, (err) => {
        errors[index] = err;
        if (--remaining === 0) reject(new AggregateError(errors, "All promises were rejected"));
      });
    });
  });
};

// Promise.prototype.finally polyfill
Promise.prototype.myFinally = function (onFinally) {
  return this.then(
    (value) => Promise.resolve(onFinally()).then(() => value),
    (reason) => Promise.resolve(onFinally()).then(() => { throw reason; }),
  );
};
```

---

# 5. Other Popular Polyfill Questions

```js
// Object.create
function objectCreate(proto, props) {
  if (proto !== null && typeof proto !== "object" && typeof proto !== "function") {
    throw new TypeError("Object prototype may only be an Object or null");
  }
  function F() {}
  F.prototype = proto;
  const obj = new F();
  if (proto === null) Object.setPrototypeOf(obj, null);
  if (props) Object.defineProperties(obj, props);
  return obj;
}

// Object.assign
function objectAssign(target, ...sources) {
  if (target == null) throw new TypeError("Cannot convert undefined or null to object");
  const to = Object(target);
  for (const src of sources) {
    if (src == null) continue;
    for (const key of Reflect.ownKeys(src)) {
      if (Object.prototype.propertyIsEnumerable.call(src, key)) to[key] = src[key];
    }
  }
  return to;
}

// Array.isArray (old-school)
const isArray = (x) => Object.prototype.toString.call(x) === "[object Array]";

// String.prototype.trim
String.prototype.myTrim = function () {
  return this.replace(/^\s+|\s+$/g, "");
};

// setInterval using setTimeout
function mySetInterval(fn, delay) {
  let id;
  const tick = () => {
    fn();
    id = setTimeout(tick, delay);
  };
  id = setTimeout(tick, delay);
  return () => clearTimeout(id); // returns a canceller
}
```

---

# 6. Interview Talking Points

- Use **regular functions** for prototype methods — arrow functions don't get `this`.
- Mention **`thisArg`**, **3-argument callbacks**, **sparse array holes**, **callback validation**.
- `reduce`: `arguments.length` to detect initial value; TypeError on empty array without one.
- `call/apply`: `Symbol` key to avoid collisions; box primitives with `Object()`; delete afterwards.
- `bind`: partial application + `new` priority + prototype preservation.
- `Promise.all`: order by index, count completions, handle empty, wrap non-promises.
- Real projects use **core-js** + Babel `preset-env` with a browserslist target — don't hand-roll.

---

# 7. Output Questions 🧪

```js
console.log([1, , 3].map((x) => x * 2).length);           // Q1
console.log([1, 2].reduce((a, b) => a + b, undefined));    // Q2
console.log([, , 1].reduce((a, b) => a + b));              // Q3
console.log(["a", "b"].map(function () { return this.x; }, { x: 1 })); // Q4
console.log(["a", "b"].map(() => this?.x, { x: 1 }));      // Q5
```

<details><summary>Answers</summary>

1. `3` — holes preserved, length unchanged.
2. `NaN` — initial value `undefined` was provided → `undefined + 1 + 2`.
3. `1` — leading holes skipped; single element → returned without calling callback.
4. `[1, 1]` — `thisArg` used by regular function.
5. `[undefined, undefined]` — arrow ignores `thisArg`.

</details>

---

## Readiness Checklist

- [ ] map, filter, reduce, forEach, find, some, every — from memory with `thisArg` + holes
- [ ] call, apply, bind (with `new`)
- [ ] Promise.all, race, allSettled, any, finally
- [ ] Explain polyfill vs transpiler
- [ ] Explain why `reduce` uses `arguments.length`
