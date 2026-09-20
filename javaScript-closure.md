# JavaScript Closures — Complete Interview & Revision Notes

> **Interview + Understanding Guide**
>
> Goal: Understand closures deeply enough to explain them in an interview and recognize where they are used in real-world JavaScript/React applications.

---

## 1. What is a Closure?

A **closure** is created when a function remembers and can access variables from its **outer lexical scope**, even after the outer function has finished executing.

### Interview Definition

> **A closure is a function bundled together with its surrounding lexical environment. It allows the function to access variables from its outer scope even after the outer function has completed execution.**

The simplest mental model:

```text
Closure = Function + Lexical Environment
```

---

# 2. First Understand Lexical Scope

Before understanding closures, we need to understand **lexical scope**.

Consider:

```js
const name = "Vipin";

function greet() {
  console.log(name);
}

greet();
```

`greet()` can access `name` because `name` exists in its outer scope.

JavaScript determines variable accessibility based on **where the function is written**, not where it is called.

This is called **lexical scoping**.

---

## 3. Nested Functions

Closures become more obvious when we have nested functions.

```js
function outer() {
  const message = "Hello";

  function inner() {
    console.log(message);
  }

  inner();
}

outer();
```

Here:

```text
outer()
 ├── message
 │
 └── inner()
       ↓
    accesses message
```

`inner()` can access `message` because `message` exists in its outer lexical scope.

---

# 4. The Real Closure Example

Now let's make one important change.

Instead of executing `inner()` immediately, we return it.

```js
function outer() {
  const message = "Hello";

  function inner() {
    console.log(message);
  }

  return inner;
}

const greet = outer();

greet();
```

At first glance, you might think:

> "How can `greet()` access `message`? `outer()` has already finished."

This is where **closure** comes into the picture.

---

# 5. What Actually Happens?

When this executes:

```js
const greet = outer();
```

`outer()` runs.

It creates:

```js
const message = "Hello";
```

Then it creates:

```js
function inner() {
  console.log(message);
}
```

Then `inner` is returned.

So:

```js
greet;
```

now points to the `inner` function.

Conceptually:

```text
greet
  ↓
inner function
  ↓
remembers
  ↓
outer lexical environment
  ↓
message = "Hello"
```

Therefore:

```js
greet();
```

can still access:

```js
message;
```

This is a closure.

---

# 6. Important: Closure Does NOT Copy the Variable

This is a very important interview concept.

Consider:

```js
function counter() {
  let count = 0;

  return function () {
    count++;
    console.log(count);
  };
}

const increment = counter();

increment();
increment();
increment();
```

Output:

```text
1
2
3
```

The closure is not creating a new copy of `count` every time.

Instead, the function maintains access to the **same lexical binding**.

Conceptually:

```text
increment
    ↓
function
    ↓
closure
    ↓
count = 0
    ↓
same binding
```

When we execute:

```js
count++;
```

the same `count` is updated.

---

# 7. Why Does the Variable Stay Alive?

Normally, we think:

```text
Function executes
      ↓
Function finishes
      ↓
Local variables disappear
```

But with closures:

```text
outer()
  ↓
creates variables
  ↓
returns inner function
  ↓
inner function still references those variables
  ↓
variables remain reachable
```

JavaScript's garbage collector can remove objects/variables when they are no longer reachable.

If a returned function still references its outer environment, that environment is still reachable.

Therefore it cannot simply be garbage collected.

---

# 8. JavaScript Engine Perspective

A function internally has a connection to the lexical environment where it was created.

Conceptually, we can think of something like:

```text
function
   │
   └── [[Environment]]
           │
           ↓
      Lexical Environment
           │
           ├── message
           ├── count
           └── other variables
```

`[[Environment]]` is a **conceptual/internal mechanism**.

It is not something you normally access like:

```js
functionName.[[Environment]]
```

That is not valid application-level JavaScript.

For interview purposes:

> A function remembers the lexical environment in which it was created.

---

# 9. Closure with a Counter

This is one of the most important closure examples.

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

Why does this work?

Because the returned function has access to:

```js
count;
```

from the `createCounter()` lexical environment.

---

# 10. Private Variables Using Closures

Closures can be used to create private state.

```js
function createBankAccount(initialBalance) {
  let balance = initialBalance;

  return {
    deposit(amount) {
      balance += amount;
    },

    getBalance() {
      return balance;
    },
  };
}

const account = createBankAccount(1000);

account.deposit(500);

console.log(account.getBalance()); // 1500
```

We cannot directly access:

```js
account.balance;
```

because `balance` is not exposed.

```js
console.log(account.balance);
```

Output:

```text
undefined
```

But the returned methods can access it through their closure.

---

# 11. Closure Creates Encapsulation

This gives us a simple form of **data privacy / encapsulation**.

```text
createBankAccount()
│
├── balance ← private
│
├── deposit() ──────┐
│                   │
└── getBalance() ───┤
                    ↓
                 closure
```

The functions have access to the private variable, while external code does not.

---

# 12. Multiple Closures Are Independent

Consider:

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
console.log(counter2()); // 2
```

Why does `counter2` start from `1`?

Because every execution of:

```js
createCounter();
```

creates a **new lexical environment**.

Conceptually:

```text
counter1
   ↓
Environment A
count = 0


counter2
   ↓
Environment B
count = 0
```

They are completely independent.

---

# 13. Closure Factory Pattern

A function that creates functions is often called a **function factory**.

Example:

```js
function createMultiplier(multiplier) {
  return function (number) {
    return number * multiplier;
  };
}

const double = createMultiplier(2);
const triple = createMultiplier(3);

console.log(double(5)); // 10
console.log(triple(5)); // 15
```

Here:

```text
double
  ↓
multiplier = 2


triple
  ↓
multiplier = 3
```

Each function remembers its own environment.

---

# 14. Closures and `var` — Classic Interview Question

Consider:

```js
for (var i = 0; i < 3; i++) {
  setTimeout(() => {
    console.log(i);
  }, 1000);
}
```

Output:

```text
3
3
3
```

Why?

Because `var` is function-scoped.

There is effectively one shared `i`.

By the time the callbacks execute:

```js
i === 3;
```

So all callbacks access the same binding.

---

# 15. `let` Solves This Differently

Now:

```js
for (let i = 0; i < 3; i++) {
  setTimeout(() => {
    console.log(i);
  }, 1000);
}
```

Output:

```text
0
1
2
```

`let` creates a new binding for each loop iteration.

Conceptually:

```text
Iteration 1 → i = 0 → callback remembers i = 0

Iteration 2 → i = 1 → callback remembers i = 1

Iteration 3 → i = 2 → callback remembers i = 2
```

This is one of the most common closure-related interview questions.

---

# 16. Closure + `setTimeout`

Consider:

```js
function startTimer() {
  const message = "Timer finished";

  setTimeout(() => {
    console.log(message);
  }, 2000);
}

startTimer();
```

Even though `startTimer()` finishes before the timer callback executes, the callback can still access:

```js
message;
```

because the callback forms a closure over its surrounding lexical environment.

---

# 17. Closures + Callbacks

Closures are heavily used with callbacks.

Example:

```js
function greetUser(name) {
  return function () {
    console.log(`Hello ${name}`);
  };
}

const greetVipin = greetUser("Vipin");

greetVipin();
```

The returned function remembers:

```js
name = "Vipin";
```

This pattern appears everywhere in JavaScript.

---

# 18. Closures + Event Listeners

Example:

```js
function setupButton(buttonId) {
  const message = "Button clicked";

  const button = document.getElementById(buttonId);

  button.addEventListener("click", () => {
    console.log(message);
  });
}

setupButton("submit");
```

The callback passed to `addEventListener` closes over:

```js
message;
```

This allows asynchronous/event-driven code to access variables from the surrounding scope.

---

# 19. Closures + Debounce

One of the most practical uses of closures is **debouncing**.

```js
function debounce(callback, delay) {
  let timer;

  return function (...args) {
    clearTimeout(timer);

    timer = setTimeout(() => {
      callback(...args);
    }, delay);
  };
}
```

Usage:

```js
const search = debounce((value) => {
  console.log("API call:", value);
}, 500);

search("r");
search("re");
search("rea");
search("react");
```

The API callback should execute only after the user stops typing.

---

## Why does debounce need a closure?

Because:

```js
let timer;
```

must survive between function calls.

Every time we call:

```js
search(...)
```

we need access to the same `timer`.

The returned function closes over:

```js
timer;
```

Conceptually:

```text
debounce()
│
├── timer
│
└── returned function
       │
       └── closure → timer
```

Without closure, the function would not be able to maintain that state between calls.

---

# 20. Closures + Throttle

Throttle also uses closures.

Example:

```js
function throttle(callback, delay) {
  let canRun = true;

  return function (...args) {
    if (!canRun) return;

    callback(...args);

    canRun = false;

    setTimeout(() => {
      canRun = true;
    }, delay);
  };
}
```

The returned function remembers:

```js
canRun;
```

between executions.

Therefore closure is helping us maintain internal state.

---

# 21. Closures + Memoization

Closures are also useful for memoization.

```js
function memoize(fn) {
  const cache = {};

  return function (value) {
    if (cache[value]) {
      return cache[value];
    }

    const result = fn(value);

    cache[value] = result;

    return result;
  };
}
```

The returned function remembers:

```js
cache;
```

through its closure.

Conceptually:

```text
memoize()
│
├── cache
│
└── returned function
       │
       └── closure → cache
```

---

# 22. Closures in React

Closures are extremely important in React.

Consider:

```jsx
function Counter() {
  const [count, setCount] = useState(0);

  function handleClick() {
    console.log(count);
  }

  return <button onClick={handleClick}>{count}</button>;
}
```

`handleClick` closes over the `count` value from that render.

This leads to an important React concept:

> **Every render creates a new function scope, and functions created during that render close over the values from that render.**

---

# 23. React Stale Closure

Consider:

```jsx
function Counter() {
  const [count, setCount] = useState(0);

  function handleClick() {
    setTimeout(() => {
      console.log(count);
    }, 3000);
  }

  return (
    <>
      <p>{count}</p>

      <button onClick={handleClick}>Show Count</button>

      <button onClick={() => setCount(count + 1)}>Increment</button>
    </>
  );
}
```

Suppose:

```text
count = 0
```

You click "Show Count".

Then immediately increase count several times.

The timeout callback may still print:

```text
0
```

Why?

Because the callback closed over the `count` value from the render in which it was created.

This is called a **stale closure**.

---

# 24. Why Does React Have Stale Closures?

Think about React renders like snapshots.

```text
Render 1
count = 0
   ↓
functions created here remember count = 0


Render 2
count = 1
   ↓
new functions remember count = 1


Render 3
count = 2
   ↓
new functions remember count = 2
```

A function created during Render 1 doesn't magically change its captured values when Render 2 happens.

It belongs to the lexical environment of Render 1.

---

# 25. Solving Stale State with Functional Updates

Consider:

```jsx
setCount(count + 1);
```

If multiple updates depend on the previous state, prefer:

```jsx
setCount((prevCount) => prevCount + 1);
```

Example:

```jsx
function incrementThreeTimes() {
  setCount((prev) => prev + 1);
  setCount((prev) => prev + 1);
  setCount((prev) => prev + 1);
}
```

This tells React:

> Give me the latest state and calculate the next state from it.

This avoids relying on a potentially stale `count` value from the current closure.

---

# 26. Closures + `useEffect`

Closures are also important when working with `useEffect`.

```jsx
useEffect(() => {
  console.log(count);
}, []);
```

The effect callback closes over the values available during the render when the effect was created.

If you expect the effect to react to changes in `count`, then:

```jsx
useEffect(() => {
  console.log(count);
}, [count]);
```

The dependency array tells React when the effect should be recreated/re-run based on the referenced reactive value.

---

# 27. Closure and `useDebounce`

Our React debounce pattern uses closures indirectly through the timer/effect lifecycle.

Example:

```jsx
function useDebounce(value, delay) {
  const [debouncedValue, setDebouncedValue] = useState(value);

  useEffect(() => {
    const timer = setTimeout(() => {
      setDebouncedValue(value);
    }, delay);

    return () => {
      clearTimeout(timer);
    };
  }, [value, delay]);

  return debouncedValue;
}
```

Here the effect callback and timeout callback capture:

```text
value
delay
timer
```

from their relevant render/effect execution.

This is why understanding closures makes React hooks much easier to understand.

---

# 28. Closure Is Not a Memory Optimization

A common misconception:

> "Closure saves memory."

Not necessarily.

Closures can actually **retain memory**.

Example:

```js
function createHandler() {
  const hugeData = new Array(1000000).fill("data");

  return function () {
    console.log(hugeData.length);
  };
}

const handler = createHandler();
```

`handler` still references `hugeData`.

Therefore the data remains reachable.

If the closure remains alive for a long time, the retained data may consume significant memory.

---

# 29. Closures and Memory Leaks

Closures themselves are not memory leaks.

The problem occurs when something remains unnecessarily reachable.

Example:

```text
Long-lived object
      ↓
event listener
      ↓
closure
      ↓
large object
```

If the event listener is never removed, the closure may continue retaining data that is no longer needed.

This can contribute to memory leaks or excessive memory retention.

---

# 30. How to Avoid Closure-Related Memory Problems

### 1. Remove event listeners

```js
function setup() {
  const handler = () => {
    console.log("clicked");
  };

  button.addEventListener("click", handler);

  return () => {
    button.removeEventListener("click", handler);
  };
}
```

### 2. Clear timers

```js
const timer = setTimeout(() => {
  // ...
}, 1000);

clearTimeout(timer);
```

### 3. Clean up subscriptions

For example:

```jsx
useEffect(() => {
  const unsubscribe = subscribe();

  return () => {
    unsubscribe();
  };
}, []);
```

### 4. Avoid unnecessarily capturing huge objects

Instead of:

```js
function createHandler(hugeObject) {
  return () => {
    console.log(hugeObject.someValue);
  };
}
```

consider whether the entire object really needs to remain reachable.

---

# 31. Common Closure Interview Question

### Question:

What will this output?

```js
function outer() {
  let x = 10;

  return function () {
    console.log(x);
  };
}

const fn = outer();

x = 20;

fn();
```

Answer:

```text
ReferenceError
```

Why?

Because:

```js
x;
```

outside the closure does not refer to the local `x`.

The `x` inside `outer()` is scoped to `outer`.

The closure still has access to its own lexical environment.

---

# 32. Another Important Question

```js
function outer() {
  let x = 10;

  return function () {
    x++;
    console.log(x);
  };
}

const fn = outer();

fn();
fn();
fn();
```

Output:

```text
11
12
13
```

Because all three calls access the same closed-over `x`.

---

# 33. Closure Interview Question — Multiple Instances

```js
function createCounter() {
  let count = 0;

  return () => ++count;
}

const a = createCounter();
const b = createCounter();

console.log(a());
console.log(a());

console.log(b());
console.log(b());
```

Output:

```text
1
2
1
2
```

Reason:

```text
a → Environment A → count
b → Environment B → count
```

Each call to `createCounter()` creates a separate environment.

---

# 34. Closure Interview Question — Loop

```js
for (var i = 0; i < 3; i++) {
  setTimeout(() => {
    console.log(i);
  }, 0);
}
```

Output:

```text
3
3
3
```

Using `let`:

```js
for (let i = 0; i < 3; i++) {
  setTimeout(() => {
    console.log(i);
  }, 0);
}
```

Output:

```text
0
1
2
```

### Interview explanation

`var` is function-scoped, while `let` is block-scoped and provides a separate binding for each loop iteration.

---

# 35. Closure vs Scope

These concepts are related but not identical.

### Scope

Scope answers:

> "Where can I access this variable?"

Example:

```js
function test() {
  const x = 10;

  console.log(x);
}
```

`x` is accessible inside `test()`.

### Closure

Closure answers:

> "How can a function continue accessing variables from an outer scope after that outer execution has finished?"

Example:

```js
function test() {
  const x = 10;

  return () => x;
}

const fn = test();

fn(); // 10
```

---

# 36. Closure vs Lexical Scope

### Lexical Scope

Determines variable accessibility based on where code is written.

### Closure

The behavior where a function retains access to its lexical environment after the outer function has finished.

Simple relationship:

```text
Lexical Scope
     ↓
determines what a function can access
     ↓
Closure
     ↓
allows that access to continue after outer execution
```

---

# 37. Real-World Uses of Closures

Closures are commonly used for:

1. Private variables
2. Data encapsulation
3. Function factories
4. Callbacks
5. Event handlers
6. `setTimeout`
7. Debouncing
8. Throttling
9. Memoization
10. Currying
11. Maintaining state
12. React event handlers
13. React hooks
14. Custom hooks
15. Module patterns

---

# 38. Closure + Currying

Closures also make currying possible.

```js
function multiply(a) {
  return function (b) {
    return a * b;
  };
}

const double = multiply(2);

console.log(double(5)); // 10
console.log(double(10)); // 20
```

The inner function remembers:

```js
a = 2;
```

through closure.

---

# 39. Closure Mental Model

Whenever you see:

```js
function outer() {
  const value = ...;

  return function inner() {
    // uses value
  };
}
```

Immediately think:

```text
inner
 ↓
closure
 ↓
remembers outer lexical environment
 ↓
can access value
```

This mental model will help you identify closures quickly.

---

# 40. How to Explain Closure in an Interview

A strong answer:

> "A closure is created when a function retains access to variables from its lexical scope even after the outer function has finished executing. In JavaScript, functions maintain a reference to the lexical environment where they were created. This allows us to implement things like private state, function factories, callbacks, debounce, throttle, and memoization."

Then show:

```js
function createCounter() {
  let count = 0;

  return function () {
    return ++count;
  };
}

const counter = createCounter();

console.log(counter()); // 1
console.log(counter()); // 2
```

Then explain:

> "`counter` closes over the `count` variable. Even though `createCounter()` has finished executing, the returned function still has access to that lexical environment."

That is a very good interview-level explanation.

---

# 41. Interview Follow-Up Questions

If the interviewer asks:

### "Why doesn't `count` get destroyed?"

Answer:

> "Because the returned function still references its lexical environment. As long as that environment remains reachable, the garbage collector cannot reclaim the referenced data."

---

### "Does closure copy variables?"

Answer:

> "No. A closure retains access to the lexical bindings. It doesn't simply make a copy of every variable."

---

### "Can closures cause memory leaks?"

Answer:

> "Closures themselves aren't memory leaks, but they can retain objects longer than necessary. If a long-lived closure references large objects, event listeners, timers, or subscriptions that aren't cleaned up, it can contribute to memory retention or leaks."

---

### "Where do you use closures in real applications?"

Answer:

> "Common examples include debounce, throttle, memoization, private state, callbacks, event handlers, function factories, and React hooks."

---

### "What is a stale closure?"

Answer:

> "A stale closure occurs when a function retains values from an older render or execution context and later executes using those outdated values. This is especially common in React with asynchronous callbacks, effects, timers, and event handlers."

---

# 42. Common Mistakes

### Mistake 1: Thinking closure means copying variables

Incorrect:

> "Closure copies the outer variables."

Better:

> "Closure retains access to the outer lexical bindings."

---

### Mistake 2: Thinking closure only exists when a function is returned

A closure can exist whenever a function accesses variables from an outer lexical scope.

Returning the function simply makes the behavior easier to demonstrate because the function survives the outer execution.

---

### Mistake 3: Thinking closure automatically causes a memory leak

Incorrect.

Closures are a normal JavaScript feature.

Memory problems happen when unnecessary objects remain reachable for too long.

---

### Mistake 4: Confusing closure with scope

Scope defines accessibility.

Closure describes a function retaining access to its surrounding lexical environment.

---

### Mistake 5: Ignoring closures in React

React developers constantly work with closures through:

```text
event handlers
useEffect
setTimeout
setInterval
callbacks
custom hooks
debounce
throttle
memoization
```

Understanding closures makes many React bugs easier to understand.

---

# 43. Closure Debugging Checklist

When you encounter unexpected behavior, ask:

```text
1. Which function created this callback?

2. Which variables does the callback reference?

3. Which render/execution did those variables come from?

4. Has the outer function already executed?

5. Is the callback asynchronous?

6. Could this be a stale closure?

7. Is the closure retaining a large object?

8. Is a timer/listener/subscription still active?

9. Does this callback need cleanup?

10. Should I use a functional state update?
```

---

# 44. One Complete Example

Let's combine several concepts.

```js
function createSearchHandler(delay) {
  let timer;

  return function (searchTerm) {
    clearTimeout(timer);

    timer = setTimeout(() => {
      console.log("Searching for:", searchTerm);
    }, delay);
  };
}

const search = createSearchHandler(500);

search("r");
search("re");
search("rea");
search("react");
```

Here we have:

```text
createSearchHandler()
        │
        ├── delay
        │
        ├── timer
        │
        └── returned function
                 │
                 └── closure
                      │
                      ├── delay
                      └── timer
```

The closure allows the returned function to maintain the `timer` between calls.

This is essentially the foundation of a debounce implementation.

---

# 45. Closure in One Diagram

```text
                    createCounter()
                           │
                           ↓
                  Lexical Environment
                  ┌─────────────────┐
                  │ count = 0       │
                  └────────┬────────┘
                           │
                           │ referenced by
                           ↓
                    returned function
                           │
                           ↓
                     ┌───────────┐
                     │ counter() │
                     └─────┬─────┘
                           │
                           ↓
                    Closure keeps
                    access to count
                           │
                    ┌──────┴──────┐
                    ↓             ↓
                  count++       count++
                    ↓             ↓
                     1             2
```

---

# 46. Final Cheat Sheet

## Definition

```text
Closure = Function + Lexical Environment
```

## Core Idea

A function can remember variables from the scope where it was created.

## Why?

Because the function retains access to its lexical environment.

## Most Common Example

```js
function outer() {
  let value = 0;

  return function () {
    return ++value;
  };
}
```

## Main Uses

```text
Private state
Function factories
Callbacks
Event handlers
Debounce
Throttle
Memoization
Currying
React hooks
React event handlers
```

## Important Concepts

```text
Lexical Scope
      ↓
Function is created
      ↓
Function references outer variables
      ↓
Function survives outer execution
      ↓
Closure
```

## Memory

```text
Closure ≠ Memory Leak

But:

Long-lived closure
      ↓
references large object
      ↓
object remains reachable
      ↓
memory retention
```

## React

```text
Each render
    ↓
creates a new lexical environment
    ↓
functions created in that render
    ↓
close over values from that render
    ↓
possible stale closure
```

## Interview One-Liner

> **A closure is a function that retains access to its lexical environment, allowing it to access variables from its outer scope even after the outer function has finished executing.**

---

# 47. Quick Interview Practice

Before an interview, make sure you can explain these without looking at the notes:

### Basic

- What is a closure?
- What is lexical scope?
- How does a closure work?
- Why does the outer variable remain accessible?

### Intermediate

- Does closure copy variables?
- Can multiple closures share state?
- Why are `counter1` and `counter2` independent?
- How does closure enable private variables?
- How does `var` vs `let` affect closure behavior?

### Advanced

- How does a closure relate to the lexical environment?
- Can closures cause memory retention?
- What is a stale closure?
- Why do stale closures happen in React?
- How does debounce use closure?
- How does throttle use closure?
- How does memoization use closure?

### React

- Why can an async callback have stale state?
- Why are functional state updates useful?
- How does `useEffect` interact with closures?
- Why does `useDebounce` work?
- Why can creating a debounced function on every render be problematic?

---

# Final Mental Model

Whenever you see:

```js
function outer() {
  let data = something();

  return function inner() {
    use(data);
  };
}
```

Think:

```text
inner()
  │
  ├── Function
  │
  └── remembers
         │
         ↓
    outer lexical environment
         │
         ↓
        data
```

That's the essence of a **JavaScript closure**.
