# JavaScript — Execution Context

## What is Execution Context?

An **Execution Context** is the environment in which JavaScript evaluates and executes code.

It keeps track of things like:

- Variables
- Functions
- Scope information
- `this`

### Types

- Global Execution Context
- Function Execution Context
- Eval Execution Context

### Creation & Execution

Conceptually:

```text
Execution Context
       │
       ├── Creation Phase
       │      ↓
       │   Prepare environment
       │
       └── Execution Phase
              ↓
           Run code
```

Example:

```js
console.log(x);

var x = 10;

console.log(x);
```

Output:

```text
undefined
10
```

`var x` is prepared as `undefined` before the assignment executes.

### Function Execution Context

Created when a function is **called**:

```js
function test() {
  var x = 20;
  console.log(x);
}

test();
```

Conceptually:

```text
Global Execution Context
        │
        ↓
     test()
        │
        ↓
Function Execution Context
        │
        └── x → 20
```

### Variable Shadowing

```js
var x = 10;

function test() {
  var x = 20;
  console.log(x);
}

test();
console.log(x);
```

Output:

```text
20
10
```

The local `x` shadows the global `x`.

## Interview Answer

> An execution context is the environment in which JavaScript code is evaluated and executed. It contains information such as variables, functions, scope and `this`. JavaScript creates a Global Execution Context for global code and a Function Execution Context whenever a function is called.

## Key Takeaways

- Execution Context is the environment in which JavaScript executes code.
- Global code runs inside the Global Execution Context.
- A function call creates a Function Execution Context.
- A function declaration does not create a Function Execution Context until the function is called.
- Execution Context can be understood conceptually through a creation/preparation phase and an execution phase.
- `var` behavior during setup helps explain hoisting.
- Local variables can shadow global variables.
- Scope and variable lookup will explain exactly how JavaScript finds variables.

# JavaScript — Global Execution Context

## What is Global Execution Context?

The **Global Execution Context (GEC)** is created when JavaScript starts executing global code.

```text
JavaScript starts
       ↓
Global Execution Context
       ↓
Global code executes
```

Example:

```js
var x = 10;

function test() {
  console.log(x);
}

test();
```

Conceptually:

```text
Global Execution Context
│
├── x → 10
└── test → function
```

When `test()` is called, a **Function Execution Context** is created, while the GEC remains available.

## Variable Lookup

```js
var x = 10;

function test() {
  console.log(x);
}

test();
```

Output:

```text
10
```

`test()` doesn't have its own `x`, so JavaScript looks outside the function and finds `x` in the global scope.

This behavior is explained by the **Scope Chain / Lexical Environment**.

## Interview Answer

> "The Global Execution Context is created when JavaScript starts executing global code. It provides the environment for global variables, functions, and other global-level execution information. When a function is called, a new Function Execution Context is created, while the Global Execution Context remains available."

## Key Takeaways

- Global code runs inside the Global Execution Context.
- The GEC is created when JavaScript starts executing global code.
- Global variables and functions are available through the global environment.
- Function Execution Contexts are created when functions are called.
- The GEC remains available while functions execute.
- Variable lookup can move from a function's scope to the global scope.
- Scope Chain and Lexical Environment explain how this lookup works.

# JavaScript — Function Execution Context

## What is Function Execution Context?

A **Function Execution Context (FEC)** is created whenever a function is **called/invoked**.

Example:

```js
function test(a) {
  const b = 20;
  console.log(a + b);
}

test(10);
```

When `test(10)` is called:

```text
Function Execution Context
│
├── a → 10
└── b → 20
```

Output:

```text
30
```

### What does FEC contain?

Conceptually, it contains information required to execute the function, including:

- Function parameters
- Local variables
- Scope-related information
- `this` value

### Accessing Outer Variables

A function can access variables from its outer scope.

```js
const x = 10;

function test(a) {
  const b = 20;
  console.log(x + a + b);
}

test(5);
```

Output:

```text
35
```

Conceptually:

```text
FEC
├── a → 5
└── b → 20

GEC
└── x → 10
```

`x` isn't found inside the FEC, so JavaScript looks in the outer environment.

This lookup is explained by the **Scope Chain**.

### FEC Lifecycle

```text
Function called
      ↓
FEC created
      ↓
Function executes
      ↓
Function returns
      ↓
FEC removed from Call Stack
```

## Interview Answer

> "A Function Execution Context is created whenever a function is invoked. It contains the function's parameters, local variables, scope-related information and `this` value. Once the function finishes execution, its execution context is removed from the call stack."

## Key Takeaways

- Function Execution Context is created when a function is called.
- Function parameters and local variables belong to the function's execution environment.
- A function can access variables from its outer scope.
- Variable lookup beyond the current function is handled through the Scope Chain.
- The FEC exists while the function is executing.
- After the function finishes, its execution context is removed from the Call Stack.

# JavaScript — Call Stack

## What is Call Stack?

The **Call Stack** keeps track of the functions currently being executed.

It follows **LIFO**:

> Last In, First Out

### Example

```js
function a() {
  console.log("A");
  b();
  console.log("A again");
}

function b() {
  console.log("B");
}

a();

console.log("Global");
```

Output:

```text
A
B
A again
Global
```

### Stack Flow

When `a()` is called:

```text
Call Stack
┌──────────┐
│ a()      │
├──────────┤
│ Global   │
└──────────┘
```

When `a()` calls `b()`:

```text
Call Stack
┌──────────┐
│ b()      │ ← executing
├──────────┤
│ a()      │
├──────────┤
│ Global   │
└──────────┘
```

`b()` finishes first, then `a()` continues.

### Function Lifecycle

```text
Function called
      ↓
Execution Context created
      ↓
Pushed onto Call Stack
      ↓
Function executes
      ↓
Function finishes
      ↓
Removed from Call Stack
```

### Stack Overflow

Infinite recursion keeps adding function calls:

```js
function test() {
  test();
}

test();
```

Eventually:

```text
Maximum call stack size exceeded
```

## Interview Answer

> "The Call Stack is a LIFO data structure used by JavaScript to keep track of function execution. When a function is called, its execution context is pushed onto the stack. When the function finishes, it is removed from the stack."

## Key Takeaways

- Call Stack keeps track of currently executing functions.
- It follows **LIFO — Last In, First Out**.
- A function's execution context is pushed onto the Call Stack when the function is called.
- The execution context is removed when the function finishes.
- Nested function calls create multiple stack frames.
- Infinite recursion can cause a **stack overflow**.

# JavaScript — Heap / Memory

## What is Heap?

The **Heap** is a region of memory used by the JavaScript engine to store dynamically allocated data, especially objects, arrays, functions, and other complex data.

Example:

```js
const user = {
  name: "Vipin",
  age: 29,
};

const numbers = [10, 20, 30];
```

Conceptually, the variables hold references to data stored in the Heap.

```text
Stack                 Heap

user ───────────────► { name: "Vipin", age: 29 }

numbers ────────────► [10, 20, 30]
```

> The exact memory representation is engine-dependent. "Objects in Heap, primitives in Stack" is a useful mental model, not a strict JavaScript specification rule.

---

## Stack vs Heap

| Call Stack                           | Heap                                      |
| ------------------------------------ | ----------------------------------------- |
| Tracks currently executing functions | Stores dynamically allocated data         |
| Uses LIFO                            | Not a LIFO data structure                 |
| Contains execution frames            | Contains objects, arrays, functions, etc. |
| Related to function execution        | Related to memory allocation              |

---

## Reference Behavior

When assigning an object to another variable:

```js
const user = {
  name: "Vipin",
};

const anotherUser = user;

anotherUser.name = "Rahul";

console.log(user.name);
```

Output:

```text
Rahul
```

Why?

Both variables reference the **same object**.

```text
user ───────────────► { name: "Rahul" }
                       ▲
anotherUser ──────────┘
```

So modifying the object through `anotherUser` also affects `user`.

---

## Primitive vs Object Assignment

### Primitive

```js
let a = 10;
let b = a;

b = 20;

console.log(a); // 10
```

The value is copied.

### Object

```js
const a = { value: 10 };
const b = a;

b.value = 20;

console.log(a.value); // 20
```

Both variables reference the same object.

---

## Creating a New Object

Using spread:

```js
const user = {
  name: "Vipin",
};

const anotherUser = { ...user };

anotherUser.name = "Rahul";

console.log(user.name); // Vipin
console.log(anotherUser.name); // Rahul
```

A new object is created:

```text
user ───────────────► { name: "Vipin" }

anotherUser ────────► { name: "Rahul" }
```

This is called a **shallow copy**.

---

## Garbage Collection

When an object is no longer reachable, the JavaScript engine can eventually remove it through **Garbage Collection (GC)**.

Example:

```js
function test() {
  const user = {
    name: "Vipin",
  };
}

test();
```

After `test()` finishes, if the object has no remaining references, it can become eligible for garbage collection.

---

## Interview Answer

**What is Heap memory in JavaScript?**

> "Heap is a region of memory where JavaScript dynamically allocates data such as objects, arrays, and functions. Variables can hold references to these objects. When objects are no longer reachable, the JavaScript engine's garbage collector can reclaim their memory."

---

## Key Takeaways

- Heap is used for dynamically allocated data such as objects, arrays, and functions.
- The exact memory layout is **engine-dependent**.
- Assigning an object to another variable copies the **reference**, not the object.
- Two variables can reference the same object.
- Modifying a shared object affects both references.
- `{ ...obj }` creates a new top-level object.
- Spread creates a **shallow copy**, not a deep copy.
- Unreachable objects can eventually be cleaned up by **Garbage Collection**.
- Don't say "all primitives are always in Stack and all objects are always in Heap" as a strict rule.
