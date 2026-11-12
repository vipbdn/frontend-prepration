# JavaScript Closures — Complete Notes

## Table of Contents

1. [Lexical Scope](#1-lexical-scope)
2. [What Is a Closure?](#2-what-is-a-closure)
3. [Lexical Environment and Scope Chain](#3-lexical-environment-and-scope-chain)
4. [How Closures Work](#4-how-closures-work)
5. [Function Factories](#5-function-factories)
6. [Closures in Event Handlers](#6-closures-in-event-handlers)
7. [Closures and Data Privacy](#7-closures-and-data-privacy)
8. [Closures in Loops: `var` vs `let`](#8-closures-in-loops-var-vs-let)
9. [Stale Closures in JavaScript](#9-stale-closures-in-javascript)
10. [Stale Closures in React](#10-stale-closures-in-react)
11. [Closures Inside `useEffect`](#11-closures-inside-useeffect)
12. [Fixing the `setInterval` Counter Bug](#12-fixing-the-setinterval-counter-bug)
13. [Common Interview Questions](#13-common-interview-questions)
14. [Key Takeaways](#14-key-takeaways)

---

## 1. Lexical Scope

### Definition

**Lexical scope** means JavaScript determines variable accessibility based on where variables and functions are defined in the source code.

A function's access to variables depends on where the function is written, not where it is called.

### Example

```js
const value = "Global";

function outer() {
  const value = "Outer";

  return function inner() {
    console.log(value);
  };
}

function run() {
  const value = "Run";
  const callback = outer();

  callback();
}

run(); // "Outer"
```

### Explanation

The `inner` function is defined inside `outer`, where `value` is `"Outer"`.

Even though `callback()` is invoked inside `run()`, the function looks for variables according to its lexical scope.

**Output:**

```text
Outer
```

### Interview definition

"Lexical scope is a scoping mechanism in which variable accessibility is determined by the location where code is defined. A function can access variables from its own scope and its enclosing scopes."

---

## 2. What Is a Closure?

### Definition

A **closure** is a function that retains access to variables from its surrounding lexical environment, even after the outer function has finished executing.

### Example

```js
function outer() {
  const name = "Vipin";

  return function inner() {
    console.log(name);
  };
}

const greet = outer();

greet(); // "Vipin"
```

### How it works

1. `outer()` creates the variable `name`.
2. `outer()` returns the `inner` function.
3. The returned function retains access to `name`.
4. Even after `outer()` finishes executing, `greet()` can still access `name`.

### Important distinction

A closure does not necessarily require returning a function.

```js
function outer() {
  const message = "Hello";

  function inner() {
    console.log(message);
  }

  inner();
}

outer(); // "Hello"
```

Here, `inner` has access to the surrounding lexical environment while it executes.

### Interview definition

"A closure is a function bundled with its lexical environment. It allows the function to retain access to variables from its surrounding scope, even when that scope's outer function has finished executing."

---

## 3. Lexical Environment and Scope Chain

### Lexical Environment

A lexical environment is an internal JavaScript mechanism that maintains identifier bindings and a reference to its outer lexical environment.

### Example

```js
const country = "India";

function outer() {
  const name = "Vipin";

  function inner() {
    const language = "JavaScript";

    console.log(language);
    console.log(name);
    console.log(country);
  }

  inner();
}

outer();
```

**Output:**

```text
JavaScript
Vipin
India
```

The `inner` function can access:

- `language` from its own environment.
- `name` from the enclosing `outer` environment.
- `country` from the global environment.

### Scope Chain

The **scope chain** is the process of searching the current lexical environment and its outer environments to resolve a variable.

JavaScript searches the nearest scope first. If the variable isn't found, it continues outward.

### Variable Shadowing

```js
const value = "Global";

function outer() {
  const value = "Outer";

  function inner() {
    const value = "Inner";
    console.log(value);
  }

  inner();
}

outer(); // "Inner"
```

The inner `value` shadows the variables with the same name in outer scopes.

### Scope Chain vs. Closure

| Concept             | Meaning                                                            |
| ------------------- | ------------------------------------------------------------------ |
| Lexical scope       | Determines where variables are accessible.                         |
| Lexical environment | Maintains identifier bindings and the outer environment reference. |
| Scope chain         | Describes how JavaScript searches environments for identifiers.    |
| Closure             | A function retains access to its surrounding lexical environment.  |

---

## 4. How Closures Work

Closures retain access to variable bindings, not simply frozen snapshots of their values.

### Example

```js
function createCounter() {
  let count = 0;

  return function () {
    count++;
    return count;
  };
}

const counter = createCounter();

console.log(counter()); // 1
console.log(counter()); // 2
console.log(counter()); // 3
```

The returned function retains access to the same `count` binding and updates it on each invocation.

### Two independent closures

```js
function createCounter() {
  let count = 0;

  return function () {
    count++;
    return count;
  };
}

const counter1 = createCounter();
const counter2 = createCounter();

console.log(counter1()); // 1
console.log(counter1()); // 2
console.log(counter2()); // 1
```

Each call to `createCounter()` creates a separate lexical environment.

Therefore, `counter1` and `counter2` maintain independent counters.

---

## 5. Function Factories

### Definition

A **function factory** is a function that creates and returns another function, often configured with initial values.

Function factories commonly use closures to preserve configuration.

### Example: Multiplier factory

```js
function createMultiplier(factor) {
  return function (number) {
    return number * factor;
  };
}

const double = createMultiplier(2);
const triple = createMultiplier(3);

console.log(double(5)); // 10
console.log(triple(5)); // 15
```

### Example: Discount calculator

```js
function createDiscountCalculator(discountPercent) {
  return function (price) {
    return price - (price * discountPercent) / 100;
  };
}

const regularCustomer = createDiscountCalculator(10);
const premiumCustomer = createDiscountCalculator(20);

console.log(regularCustomer(1000)); // 900
console.log(premiumCustomer(1000)); // 800
```

### Why it works

Each factory invocation creates its own `discountPercent` binding. The returned function retains access to that binding through a closure.

### Interview answer

"A function factory creates and returns a function. The returned function can use configuration values from the factory's lexical environment through a closure, making it useful for reusable and configurable behavior."

---

## 6. Closures in Event Handlers

Event handlers often execute later, after the function that registered them has finished executing.

Closures allow these handlers to retain access to variables from their surrounding scope.

### Example: Click counter

```js
function setupCounter() {
  let count = 0;

  document.querySelector("button").addEventListener("click", function () {
    count++;
    console.log(`Clicked ${count} times`);
  });
}

setupCounter();
```

Each button click increments the same `count` binding.

**Expected output across three clicks:**

```text
Clicked 1 times
Clicked 2 times
Clicked 3 times
```

### Interview answer

"An event handler can form a closure over variables in its surrounding lexical environment. This allows it to access or update those variables when an event occurs later."

---

## 7. Closures and Data Privacy

Closures can be used to encapsulate data and expose only specific operations.

### Example: Bank account

```js
function createBankAccount() {
  let balance = 1000;

  return {
    deposit(amount) {
      balance += amount;
      return balance;
    },

    getBalance() {
      return balance;
    },
  };
}

const account = createBankAccount();

console.log(account.getBalance()); // 1000
console.log(account.deposit(500)); // 1500
console.log(account.balance); // undefined
```

The `balance` variable is local to `createBankAccount`. It isn't a property of the returned object.

The returned methods can access and modify it through closures.

### Independent accounts

```js
const account1 = createBankAccount();
const account2 = createBankAccount();

account1.deposit(500);

console.log(account1.getBalance()); // 1500
console.log(account2.getBalance()); // 1000
```

Each account has its own private `balance` binding.

**Note:** Closures provide encapsulation, not a complete security boundary.

---

## 8. Closures in Loops: `var` vs `let`

### The problem with `var`

```js
for (var i = 1; i <= 3; i++) {
  setTimeout(function () {
    console.log(i);
  }, 1000);
}
```

**Output:**

```text
4
4
4
```

### Why?

- `var` is function-scoped, not block-scoped.
- The callbacks share the same `i` binding.
- The loop finishes before the callbacks execute.
- At that point, `i` is `4`.

Each callback reads the same final value.

### Solution 1: Use `let`

```js
for (let i = 1; i <= 3; i++) {
  setTimeout(function () {
    console.log(i);
  }, 1000);
}
```

**Output:**

```text
1
2
3
```

With `let` in the loop initializer, each iteration has its own binding for the loop variable.

### Solution 2: Use an IIFE

An Immediately Invoked Function Expression creates a new function scope for each iteration.

```js
for (var i = 1; i <= 3; i++) {
  (function (capturedI) {
    setTimeout(function () {
      console.log(capturedI);
    }, 1000);
  })(i);
}
```

**Output:**

```text
1
2
3
```

Each IIFE invocation receives the current `i` as its `capturedI` parameter.

### Solution 3: Use a helper function

```js
function createLogger(value) {
  setTimeout(function () {
    console.log(value);
  }, 1000);
}

for (var i = 1; i <= 3; i++) {
  createLogger(i);
}
```

Each call creates a separate invocation with its own `value` parameter.

**Interview takeaway:** `let` is generally the simplest solution in modern JavaScript. IIFEs and helper functions can solve the problem when `var` must be retained.

---

## 9. Stale Closures in JavaScript

### Definition

A stale closure occurs when a callback reads an older value or snapshot than expected.

Closures themselves aren't inherently stale. The issue depends on what the callback reads and when that value was computed.

### Example

```js
let count = 0;

function createLogger() {
  const message = `Count is ${count}`;

  return function () {
    console.log(message);
  };
}

const logCount = createLogger();

count = 5;

logCount(); // "Count is 0"
```

### Why?

The `message` string was computed when `count` was `0`.

Changing `count` later doesn't automatically recompute the existing string.

### Fix: Read the binding when the callback runs

```js
let count = 0;

function createLogger() {
  return function () {
    console.log(`Count is ${count}`);
  };
}

const logCount = createLogger();

count = 5;

logCount(); // "Count is 5"
```

The callback reads the current value of the same `count` binding when it executes.

### Important distinction

Closures retain access to variable bindings. A computed value, such as a string or number stored in another variable, does not automatically update when its original input changes.

---

## 10. Stale Closures in React

React state is associated with individual renders. Every render creates new functions that see that render's state snapshot.

### Example

```jsx
import { useState } from "react";

function Counter() {
  const [count, setCount] = useState(0);

  function handleClick() {
    setTimeout(() => {
      console.log(count);
    }, 2000);
  }

  return (
    <>
      <p>Count: {count}</p>

      <button onClick={handleClick}>Log count after 2 seconds</button>

      <button onClick={() => setCount((c) => c + 1)}>Increment</button>
    </>
  );
}
```

### Reproduction steps

1. Initial count is `0`.
2. Click **Log count after 2 seconds**.
3. Increment the counter twice.
4. The screen displays `2`.
5. When the timer fires, the console logs `0`.

### Why?

The timer callback closes over the `count` from the render in which `handleClick` ran.

A later render creates new functions with updated state, but it doesn't change the state snapshot referenced by the already-created callback.

### Fix 1: Functional state updates

When calculating new state from previous state, use a functional updater.

```jsx
setCount((previousCount) => previousCount + 1);
```

This avoids relying on a potentially stale `count` value for the update.

### Fix 2: Use a ref for latest-value reads

```jsx
import { useState, useRef, useEffect } from "react";

function Counter() {
  const [count, setCount] = useState(0);
  const countRef = useRef(count);

  useEffect(() => {
    countRef.current = count;
  }, [count]);

  function handleClick() {
    setTimeout(() => {
      console.log(countRef.current);
    }, 2000);
  }

  return (
    <>
      <p>Count: {count}</p>
      <button onClick={handleClick}>Log count later</button>
      <button onClick={() => setCount((c) => c + 1)}>Increment</button>
    </>
  );
}
```

The timer reads `countRef.current`, which is updated after each committed render.

Use refs when a long-lived callback needs to read the latest value without being recreated on every render.

---

## 11. Closures Inside `useEffect`

`useEffect` runs after React commits a render. The effect's callback closes over values from that render.

### Example

```jsx
useEffect(() => {
  console.log(count);
}, []);
```

With an empty dependency array, the effect normally runs once after the initial mount (development Strict Mode can perform an additional setup cycle).

The effect reads the initial `count` value.

If you intend the effect to run when `count` changes, include it in the dependency array:

```jsx
useEffect(() => {
  console.log(count);
}, [count]);
```

React will run the effect again when `count` changes, cleaning up the previous effect first if a cleanup function was returned.

### Dependency array rules

| Dependency array    | Behavior                                                           |
| ------------------- | ------------------------------------------------------------------ |
| No dependency array | Effect runs after every committed render.                          |
| `[]`                | Effect runs on mount, subject to development Strict Mode behavior. |
| `[count]`           | Effect re-runs when `count` changes.                               |

**Important:** Include the reactive values that an effect uses in its dependency array. Don't omit dependencies merely to force an effect to run once.

---

## 12. Fixing the `setInterval` Counter Bug

### The classic bug

```jsx
import { useState, useEffect } from "react";

function Counter() {
  const [count, setCount] = useState(0);

  useEffect(() => {
    const timer = setInterval(() => {
      setCount(count + 1);
    }, 1000);

    return () => clearInterval(timer);
  }, []);

  return <h1>Count: {count}</h1>;
}
```

**Problem:** The interval callback closes over the initial `count = 0`.

It repeatedly calls `setCount(1)`, so the counter becomes `1` and stays there.

### Fix 1: Functional state updater

This is the recommended solution when the next state depends on the previous state.

```jsx
useEffect(() => {
  const timer = setInterval(() => {
    setCount((previousCount) => previousCount + 1);
  }, 1000);

  return () => clearInterval(timer);
}, []);
```

**Result:**

```text
0 → 1 → 2 → 3 → 4 → 5 ...
```

React supplies the previous state to the updater function when processing the update.

### Fix 2: Add `count` to dependencies

```jsx
useEffect(() => {
  const timer = setInterval(() => {
    setCount(count + 1);
  }, 1000);

  return () => clearInterval(timer);
}, [count]);
```

When `count` changes, React cleans up the previous interval and creates another with the latest `count`.

This works, but the interval is recreated on each increment.

### Fix 3: Use a ref

A ref can store the latest committed value for a long-lived callback.

```jsx
const countRef = useRef(count);

useEffect(() => {
  countRef.current = count;
}, [count]);

useEffect(() => {
  const timer = setInterval(() => {
    console.log(countRef.current);
  }, 1000);

  return () => clearInterval(timer);
}, []);
```

Use this approach when the callback needs to read the latest value. For a simple counter update, the functional updater is simpler.

### Which fix should you choose?

| Requirement                                      | Preferred approach |
| ------------------------------------------------ | ------------------ |
| Increment state based on previous state          | Functional updater |
| Re-run an effect when a value changes            | Dependency array   |
| Read the latest value from a long-lived callback | Ref                |

### Interview answer

"The interval callback closes over the initial render's state snapshot. With an empty dependency array, that callback continues to use the initial value. I would use a functional state update, such as `setCount(previousCount => previousCount + 1)`, because it calculates the next value from React's previous state without depending on a captured `count`."

---

## 13. Common Interview Questions

### Q1. What is a closure?

A closure is a function that retains access to variables from its surrounding lexical environment.

### Q2. What is lexical scope?

Lexical scope determines variable accessibility based on where code is defined.

### Q3. What is the difference between scope chain and closure?

The scope chain describes how JavaScript resolves variable names. A closure describes a function retaining access to its surrounding lexical environment.

### Q4. What is a function factory?

A function that creates and returns another function, often configured with initial values.

### Q5. How do closures help with data privacy?

Closures allow methods to access local variables that aren't directly exposed as properties.

### Q6. Why does a `var` loop sometimes print the same value in asynchronous callbacks?

The callbacks share the same function-scoped `var` binding and execute after the loop has changed its value.

### Q7. What is a stale closure in React?

A callback reads state or props from an earlier render instead of the newer values expected by the developer.

### Q8. How do you fix a stale closure in `setInterval`?

Use a functional state update when the next state depends on the previous state. Use effect dependencies or refs when the actual requirement calls for them.

### Q9. Does a closure capture a copy of every variable?

Not necessarily. A closure retains access to its lexical environment and bindings. A separately computed primitive value, such as a string, does not automatically recalculate when its inputs change.

### Q10. Are all callbacks asynchronous?

No. A callback can execute synchronously or asynchronously, depending on how the receiving function invokes it.

---

## 14. Key Takeaways

- Lexical scope determines where variables can be accessed.
- A closure retains access to its surrounding lexical environment.
- Closures can preserve mutable bindings across function calls.
- Function factories use closures to create configurable functions.
- Event handlers use closures to access surrounding variables when events occur.
- `var` loop callbacks can share one binding, while `let` provides per-iteration bindings.
- A stale closure occurs when a callback reads an older value or snapshot than expected.
- React functions see the state snapshot associated with their render.
- Use functional state updates when new state depends on previous state.
- Use effect dependencies to synchronize effects with reactive values.
- Use refs when long-lived callbacks need to read the latest committed value.
- Always clean up intervals, timers, and event listeners when their lifecycle ends.

**Final interview principle:** Understand which lexical environment a callback accesses, when its values are computed, and whether it needs a state update, a new effect, or access to the latest value. That reasoning helps you diagnose most closure-related bugs in JavaScript and React.
