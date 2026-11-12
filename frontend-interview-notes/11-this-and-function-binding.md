# 11 — `this` & Function Binding 🔥🔥🔥

> **Interview + Understanding Guide** — `00-starter-roadmap.md` Section 11
>
> Goal: Determine the value of `this` in any situation in seconds, understand `call`/`apply`/`bind` deeply, and implement all three from scratch.

---

# 1. What is `this`? ⭐⭐⭐

`this` is a special keyword whose value is the **object the function is operating on** — and for regular functions it's decided **at call time, by how the function is called**, not where it's defined.

> **Key insight:** For regular functions, ask **"how was it called?"**. For arrow functions, ask **"where was it written?"**.

### Interview Definition

> **`this` is a context value set when a function is invoked. For regular functions it depends on the call site: method call, plain call, `new`, or explicit binding with `call/apply/bind`. Arrow functions don't have their own `this`; they use `this` from the surrounding lexical scope.**

---

# 2. The Rules (in priority order) ⭐⭐⭐

```text
Is it an arrow function?            → this = enclosing scope's this (lexical). STOP.
Called with `new`?                  → this = the newly created object
Called with call / apply / bind?    → this = the given object
Called as obj.method()?             → this = obj (the object before the dot)
Plain call fn()?                    → strict: undefined | non-strict: globalThis
```

| Call | `this` |
| --- | --- |
| `new Foo()` | new instance |
| `fn.call(obj)` / `fn.apply(obj)` / `fn.bind(obj)()` | `obj` |
| `obj.fn()` | `obj` |
| `fn()` (strict / module / class) | `undefined` |
| `fn()` (sloppy script) | `globalThis` (`window` in browsers) |
| Arrow function | inherited from outer scope, **can't be changed** |

Priority: **`new` > explicit (`bind`) > implicit (method call) > default**.

---

# 3. `this` in Each Context

## 3.1 Global scope

```js
console.log(this);
```

| Environment | Value |
| --- | --- |
| Browser classic `<script>` | `window` |
| ES module (`type="module"`) | `undefined` |
| Node CommonJS file | `module.exports` (`{}`) |

## 3.2 Inside an object method (implicit binding)

```js
const user = {
  name: "Asha",
  greet() {
    return `Hi ${this.name}`;
  },
};

user.greet(); // "Hi Asha" — this = user (object before the dot)
```

Only the **last** object before the dot counts:

```js
const company = {
  name: "Acme",
  ceo: { name: "Ravi", getName() { return this.name; } },
};
company.ceo.getName(); // "Ravi"
```

## 3.3 Inside a regular function (default binding)

```js
function show() {
  return this;
}

show(); // strict: undefined | sloppy: window/globalThis
```

## 3.4 Inside an arrow function (lexical `this`)

Arrows **don't have their own `this`** — they look it up like a normal variable in the enclosing scope.

```js
const user = {
  name: "Asha",
  greetArrow: () => this.name, // ❌ this = outer scope (module: undefined → TypeError; script: window)
  greet() {
    const inner = () => this.name; // ✅ inherits greet's this
    return inner();
  },
};

user.greet(); // "Asha"
```

> ❗ **Object literals don't create a scope for `this`.** An arrow defined as an object property gets the `this` of the code **around the object**.

Arrow functions also:
- Ignore `call`/`apply`/`bind` for `this`
- Can't be used with `new`
- Have no `arguments` object

## 3.5 Inside a class

```js
class Counter {
  count = 0;

  increment() {
    this.count++;
  }

  incrementArrow = () => {
    this.count++; // class field arrow → this permanently = instance
  };
}

const c = new Counter();
c.increment();        // ✅ this = c

const fn = c.increment;
fn();                 // ❌ TypeError: Cannot read properties of undefined — classes are strict

const fn2 = c.incrementArrow;
fn2();                // ✅ works
```

## 3.6 With `new`

```js
function Person(name) {
  this.name = name; // this = brand-new object
}
const p = new Person("Asha");
```

## 3.7 In DOM event handlers

```js
button.addEventListener("click", function () {
  console.log(this); // the element (= event.currentTarget)
});

button.addEventListener("click", () => {
  console.log(this); // outer scope's this, NOT the button
});
```

---

# 4. Losing `this` ⭐⭐⭐ (the most common bug)

Taking a method **off** its object and calling it separately → the "object before the dot" is gone.

```js
const user = {
  name: "Asha",
  greet() {
    console.log(this.name);
  },
};

const greet = user.greet;
greet();                    // undefined / TypeError — plain call

setTimeout(user.greet, 0);  // ❌ same problem — setTimeout calls it as a plain function
[1].forEach(user.greet);    // ❌
button.addEventListener("click", user.greet); // ❌ this = button
```

### Fixes

```js
// 1. Wrap in an arrow (call it as a method)
setTimeout(() => user.greet(), 0);

// 2. bind
setTimeout(user.greet.bind(user), 0);

// 3. Class field arrow functions
class User {
  name = "Asha";
  greet = () => console.log(this.name);
}

// 4. Many array methods accept a thisArg
[1].forEach(user.greet, user);
```

### Callbacks inside methods

```js
const team = {
  name: "Core",
  members: ["Asha", "Ravi"],
  list() {
    // ❌ regular function callback: this = undefined
    // return this.members.map(function (m) { return `${m} - ${this.name}`; });

    // ✅ arrow inherits list()'s this
    return this.members.map((m) => `${m} - ${this.name}`);
  },
};
```

Old-school fix: `const self = this;` before the callback.

---

# 5. `call`, `apply`, `bind` ⭐⭐⭐

All three let you **set `this` explicitly**.

```js
function introduce(greeting, punctuation) {
  return `${greeting}, I'm ${this.name}${punctuation}`;
}

const asha = { name: "Asha" };

introduce.call(asha, "Hi", "!");      // args one by one   → invokes now
introduce.apply(asha, ["Hi", "!"]);   // args as an array  → invokes now
const bound = introduce.bind(asha, "Hi"); // returns a NEW function, doesn't invoke
bound("!");                           // "Hi, I'm Asha!"
```

| | Invokes immediately? | Arguments | Returns |
| --- | --- | --- | --- |
| `call` | ✅ | comma-separated | function's result |
| `apply` | ✅ | array (or array-like) | function's result |
| `bind` | ❌ | comma-separated (partially applied) | **new bound function** |

> Memory trick: **C**all = **C**ommas, **A**pply = **A**rray, **B**ind = **B**ack later.

### `bind` details

- `bind` also does **partial application** (pre-fills arguments).
- A bound function's `this` **can't be changed again** — re-binding or `call` won't override it.
- **`new` overrides `bind`**: `new BoundFn()` ignores the bound `this` (but keeps pre-filled args).

```js
const f = function () { return this.name; }.bind({ name: "first" });
f.call({ name: "second" });         // "first"
f.bind({ name: "third" })();        // "first"
```

### Method borrowing

```js
// Use Array methods on array-like objects
function sum() {
  return Array.prototype.reduce.call(arguments, (a, b) => a + b, 0);
}
sum(1, 2, 3); // 6

const nodes = document.querySelectorAll("li");
Array.prototype.map.call(nodes, (li) => li.textContent);

// apply to spread an array (pre-ES6)
Math.max.apply(null, [3, 7, 2]); // 7 — today: Math.max(...arr)

// Borrow a method from another object
const ravi = { name: "Ravi" };
introduce.call(ravi, "Hello", ".");
```

---

# 6. Implementations ⭐⭐⭐

**Core trick:** temporarily attach the function to the target object and call it as a **method** → `this` becomes that object. Use a `Symbol` key so we never overwrite an existing property.

## 6.1 `myCall`

```js
Function.prototype.myCall = function (context, ...args) {
  // null/undefined → globalThis (sloppy behaviour); primitives → wrapper objects
  context = context === null || context === undefined ? globalThis : Object(context);

  const key = Symbol("fn");  // unique key — no collision
  context[key] = this;       // `this` here = the function myCall was called on

  const result = context[key](...args); // method call → this = context
  delete context[key];                  // clean up
  return result;
};

function greet(g) {
  return `${g}, ${this.name}`;
}
greet.myCall({ name: "Asha" }, "Hi"); // "Hi, Asha"
```

## 6.2 `myApply`

```js
Function.prototype.myApply = function (context, args = []) {
  context = context === null || context === undefined ? globalThis : Object(context);

  const key = Symbol("fn");
  context[key] = this;
  const result = context[key](...args);
  delete context[key];
  return result;
};

greet.myApply({ name: "Ravi" }, ["Hello"]); // "Hello, Ravi"
```

## 6.3 `myBind` (basic)

```js
Function.prototype.myBind = function (context, ...boundArgs) {
  const fn = this;
  return function (...callArgs) {
    return fn.apply(context, [...boundArgs, ...callArgs]); // partial application
  };
};
```

## 6.4 `myBind` (complete — supports `new`) ⭐

```js
Function.prototype.myBind = function (context, ...boundArgs) {
  if (typeof this !== "function") {
    throw new TypeError("myBind must be called on a function");
  }
  const fn = this;

  function bound(...callArgs) {
    // Called with `new`? Then `this` is an instance of `bound` → ignore context
    const isNew = this instanceof bound;
    return fn.apply(isNew ? this : context, [...boundArgs, ...callArgs]);
  }

  // Instances created with `new bound()` should inherit from fn.prototype
  if (fn.prototype) {
    bound.prototype = Object.create(fn.prototype);
  }

  return bound;
};

function Point(x, y) {
  this.x = x;
  this.y = y;
}
const BoundPoint = Point.myBind(null, 1);
const p = new BoundPoint(2);
p.x, p.y;               // 1, 2
p instanceof Point;     // true
```

### Talking points

- Why a `Symbol`? Avoid overwriting a real property named `fn` on the object.
- Why `Object(context)`? `call` with a primitive (`greet.call("hi")`) boxes it in sloppy mode; you can't add properties to a primitive.
- Strict mode note: native `call(null)` in strict functions keeps `this = null`; the polyfill uses `globalThis` for simplicity — mention this.
- `bind` + `new`: `new` takes priority over bound `this`.

---

# 7. `this` in React

```jsx
// Class components — the classic problem
class Toggle extends React.Component {
  state = { on: false };

  // ❌ Regular method passed as callback → `this` is undefined when React calls it
  handleClickBroken() {
    this.setState({ on: !this.state.on });
  }

  // ✅ Option 1: bind in constructor
  constructor(props) {
    super(props);
    this.handleClickBroken = this.handleClickBroken.bind(this);
  }

  // ✅ Option 2: class field arrow (most common)
  handleClick = () => this.setState((s) => ({ on: !s.on }));

  render() {
    return <button onClick={this.handleClick}>{String(this.state.on)}</button>;
  }
}
```

Function components and hooks **don't use `this` at all** — one reason hooks replaced classes. Closures replace `this` (and bring stale-closure issues instead).

---

# 8. Output Questions 🧪

### Q1

```js
const obj = {
  name: "A",
  regular() { return this.name; },
  arrow: () => this?.name,
};
console.log(obj.regular());
console.log(obj.arrow());
```

<details><summary>Answer</summary>

`A`, then `undefined` (module/Node) — arrow's `this` is the outer scope, not `obj`. (In a browser classic script it would be `window.name`, i.e. `""`.)

</details>

### Q2

```js
"use strict";
const user = {
  name: "Asha",
  greet() { return this.name; },
};
const g = user.greet;
console.log(g());
```

<details><summary>Answer</summary>

`TypeError: Cannot read properties of undefined (reading 'name')` — plain call in strict mode → `this = undefined`.

</details>

### Q3

```js
const user = {
  name: "Asha",
  greet() {
    setTimeout(function () { console.log(this?.name); }, 0);
    setTimeout(() => console.log(this.name), 0);
  },
};
user.greet();
```

<details><summary>Answer</summary>

First: not `"Asha"` (`undefined` in Node, `""`/`window.name` in browsers) — regular callback, called by the timer. Second: `"Asha"` — arrow inherits `greet`'s `this`.

</details>

### Q4

```js
function f() { return this.x; }
const a = { x: 1, f };
const b = { x: 2, f: a.f };
const c = { x: 3, f: f.bind(a) };
console.log(a.f(), b.f(), c.f(), c.f.call(b));
```

<details><summary>Answer</summary>

`1 2 1 1` — implicit binding uses the object before the dot; bound functions can't be re-bound, even with `call`.

</details>

### Q5

```js
const counter = {
  count: 0,
  inc() { this.count++; return this; },
};
console.log(counter.inc().inc().count);
```

<details><summary>Answer</summary>

`2` — returning `this` enables method chaining.

</details>

### Q6

```js
function Foo() {
  this.a = 1;
  const arrow = () => this.a;
  return { a: 2, arrow };
}
const o = new Foo();
console.log(o.arrow());
```

<details><summary>Answer</summary>

`1` — the arrow captured `this` of the constructor call (the original new object with `a: 1`), even though a different object was returned.

</details>

### Q7

```js
const obj = {
  name: "outer",
  inner: {
    name: "inner",
    get() { return this.name; },
  },
};
const fn = obj.inner.get;
console.log(obj.inner.get(), (obj.inner.get)(), (0, obj.inner.get)?.call(obj));
```

<details><summary>Answer</summary>

`inner inner outer` — parentheses around a member expression keep the reference (still a method call); the comma operator produces a plain function value, then `call(obj)` sets `this` explicitly.

</details>

### Q8

```js
const bound = function () { return this.v; }.bind({ v: 1 });
const Ctor = function (v) { this.v = v; }.bind({ v: "ignored" });
console.log(bound.call({ v: 2 }), new Ctor(3).v);
```

<details><summary>Answer</summary>

`1 3` — bound `this` beats `call`; `new` beats bound `this`.

</details>

---

# 9. Interview Answers

**How is `this` determined?**

> "For regular functions it's determined by how the function is called: with `new` it's the new object; with `call`, `apply` or `bind` it's the object passed; as `obj.method()` it's `obj`; and as a plain call it's `undefined` in strict mode or the global object otherwise. Arrow functions don't have their own `this` — they use the `this` of the scope where they're defined, and that can't be changed."

**call vs apply vs bind**

> "All three set `this` explicitly. `call` invokes immediately with arguments listed individually, `apply` invokes immediately with an array of arguments, and `bind` returns a new function with `this` (and optionally some arguments) permanently bound, to be called later."

**Why do methods lose `this`?**

> "Because `this` is set by the call site. When you pass `obj.method` as a callback, you pass only the function; whoever calls it later calls it as a plain function, so `this` becomes `undefined` or the global object. Fix it with an arrow wrapper, `bind`, or a class-field arrow function."

---

# 10. Key Takeaways

- Regular function: `this` = how it's called. Arrow: `this` = where it's written.
- Priority: `new` > `bind/call/apply` > `obj.method()` > default.
- Strict mode/classes/modules: default `this` is `undefined`.
- Object literals don't create a `this` scope for arrow properties.
- Passing a method as a callback loses `this` → arrow wrapper / `bind` / class-field arrow.
- `bind` returns a permanently bound function (except under `new`).
- Implement `call/apply` by temporarily attaching the function under a `Symbol` key.

## Readiness Checklist

- [ ] State the 5 rules in priority order
- [ ] Predict `this` for methods, plain calls, arrows, classes, events, timers
- [ ] Explain and fix "losing this" 3 ways
- [ ] Implement `myCall`, `myApply`, `myBind` (with `new` support)
- [ ] Solve all 8 output questions
