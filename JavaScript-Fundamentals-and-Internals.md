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

# JavaScript — Lexical Environment & Scope Chain

## What is Lexical Environment?

A **Lexical Environment** is an internal structure JavaScript uses to keep track of variables/functions and their relationship with the surrounding scope.

Think:

> "Where are my variables, and where should I look if I can't find one here?"

Example:

```js
const x = 10;

function test() {
  const y = 20;

  console.log(x);
  console.log(y);
}

test();
```

Conceptually:

```text
Global Lexical Environment
├── x → 10
└── test → function
       │
       ▼
Function Lexical Environment
└── y → 20
```

---

## Execution Context vs Lexical Environment

These concepts are related but not the same.

| Execution Context                                                                     | Lexical Environment                                                          |
| ------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------- |
| Represents the overall environment in which code executes                             | Manages identifiers such as variables/functions and scope relationships      |
| Concerned with **code execution**                                                     | Concerned with **variable lookup and scope**                                 |
| Includes information needed for execution, such as `this` and environment information | Contains variable/function bindings and a reference to the outer environment |
| Example: Function Execution Context                                                   | Example: `test()` Lexical Environment                                        |

### Easy Mental Model

```text
Execution Context
    ↓
"How is this code being executed?"

Lexical Environment
    ↓
"Where are my variables/functions,
and where should I look if I can't find one here?"
```

> This is a practical interview mental model. ECMAScript specification terminology is more nuanced.

---

## Outer Environment Reference

A Lexical Environment has a relationship/reference to its outer environment.

```text
inner Lexical Environment
        │
        ▼
outer Lexical Environment
        │
        ▼
Global Lexical Environment
```

This relationship allows JavaScript to search for variables outside the current scope.

---

# Scope Chain

## What is Scope Chain?

The **Scope Chain** is the chain of Lexical Environments JavaScript searches when looking for a variable.

Example:

```js
const x = 10;

function test() {
  const y = 20;

  function inner() {
    const z = 30;

    console.log(x);
    console.log(y);
    console.log(z);
  }

  inner();
}

test();
```

Conceptually:

```text
inner Lexical Environment
        ↓
test Lexical Environment
        ↓
Global Lexical Environment
```

---

## Variable Lookup

JavaScript searches from the **current environment outward**.

For example:

```js
function outer() {
  const x = 20;

  function inner() {
    console.log(x);
  }

  inner();
}
```

When `inner()` accesses `x`:

```text
1. inner environment → x? ❌
2. outer environment → x? ✅
3. Stop searching
```

JavaScript stops as soon as it finds the variable.

---

## Shadowing

An inner variable can have the same name as an outer variable.

```js
const x = 10;

function test() {
  const x = 20;

  function inner() {
    const x = 30;

    console.log(x);
  }

  inner();
}

test();
```

Output:

```text
30
```

Lookup:

```text
inner → x = 30 ✅
test  → x = 20
global → x = 10
```

The closest matching variable is used.

---

## Scope Lookup Direction

Variable lookup generally follows:

```text
Current Scope
     ↓
Outer Scope
     ↓
Global Scope
```

It does **not** search from outer scope into unrelated inner scopes.

Example:

```js
function test() {
  console.log(x);
}

function another() {
  const x = 20;
}

test();
```

`test()` cannot access `x` from `another()` because `another()` is not its lexical parent.

---

## Lexical Scope

JavaScript uses **lexical scoping**.

This means a function's accessible outer variables are determined by **where the function is written**, not where it is called.

```js
const x = "global";

function outer() {
  const x = "outer";

  function inner() {
    console.log(x);
  }

  return inner;
}

const fn = outer();

fn(); // outer
```

`inner()` was written inside `outer()`, so its scope chain includes `outer()`.

This concept is the foundation of **Closures**.

---

## Interview Answer — Lexical Environment

> "A Lexical Environment is an internal structure used by JavaScript to store variable and function bindings and maintain a reference to the outer environment. This allows JavaScript to resolve variables through the scope chain."

## Interview Answer — Scope Chain

> "The Scope Chain is the chain of lexical environments JavaScript searches when resolving a variable. It starts from the current scope and moves outward until the variable is found or the global scope is reached."

---

## Key Takeaways

- Lexical Environment manages variable/function bindings and scope relationships.
- Execution Context represents the overall environment in which code executes.
- A Lexical Environment has a relationship/reference to its outer environment.
- Scope Chain is the process/path used for variable lookup through these environments.
- Variable lookup goes **inner → outer → global**.
- JavaScript stops searching when it finds the variable.
- Inner variables can **shadow** outer variables.
- JavaScript uses **lexical scoping**.
- A function's scope is determined by **where it is written**, not where it is called.
- Lexical Environment + Scope Chain are fundamental to understanding **Closures**.

# JavaScript — Scope & Environments

## 1. Lexical Environment

A **Lexical Environment** is an internal structure JavaScript uses to keep track of variable/function bindings and their relationship with the surrounding scope.

Think:

> "Where are my variables, and where should I look if I can't find one here?"

Example:

```js
const x = 10;

function test() {
  const y = 20;

  console.log(x);
  console.log(y);
}

test();
```

Conceptually:

```text
Global Lexical Environment
├── x → 10
└── test → function
       │
       ▼
Function Lexical Environment
└── y → 20
```

---

# 2. Execution Context vs Lexical Environment

These concepts are related but not the same.

| Execution Context                                                                     | Lexical Environment                                                          |
| ------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------- |
| Represents the overall environment in which code executes                             | Manages identifiers and scope relationships                                  |
| Concerned with code execution                                                         | Concerned with variable lookup and scope                                     |
| Includes information needed for execution, such as `this` and environment information | Contains variable/function bindings and a reference to the outer environment |
| Example: Function Execution Context                                                   | Example: `test()` Lexical Environment                                        |

### Easy Mental Model

```text
Execution Context
    ↓
"How is this code being executed?"

Lexical Environment
    ↓
"Where are my variables/functions,
and where should I look if I can't find one here?"
```

> This is a practical interview mental model. ECMAScript specification terminology is more nuanced.

---

# 3. Variable Environment

**Variable Environment** is a specification concept used to describe how variable declarations, especially `var`, are handled within an execution environment.

A useful interview mental model:

```text
var       → Function scoped
let/const → Block scoped
```

Example:

```js
function test() {
  var a = 10;
  let b = 20;
  const c = 30;
}
```

Conceptually:

```text
test() Environment
├── var a → 10
└── lexical bindings
    ├── b → 20
    └── c → 30
```

> Don't treat Variable Environment and Lexical Environment as completely separate physical memory areas. They are specification concepts; JavaScript engines can implement them differently.

---

# 4. Scope Chain

The **Scope Chain** is the chain of environments JavaScript searches when resolving a variable.

Example:

```js
const x = 10;

function test() {
  const y = 20;

  function inner() {
    const z = 30;

    console.log(x);
    console.log(y);
    console.log(z);
  }

  inner();
}

test();
```

Lookup:

```text
inner()
   ↓
test()
   ↓
Global
```

JavaScript searches from the **current environment outward**.

If a variable is found, the search stops.

---

# 5. Variable Shadowing

An inner scope can declare a variable with the same name as an outer scope.

```js
const x = 10;

function test() {
  const x = 20;

  function inner() {
    const x = 30;

    console.log(x);
  }

  inner();
}

test();
```

Output:

```text
30
```

Lookup:

```text
inner → x = 30 ✅
test  → x = 20
global → x = 10
```

The closest matching variable is used.

---

# 6. Lexical Scoping

JavaScript uses **lexical scoping**.

A function's accessible outer variables are determined by **where the function is written**, not where it is called.

```js
const x = "global";

function outer() {
  const x = "outer";

  function inner() {
    console.log(x);
  }

  return inner;
}

const fn = outer();

fn(); // "outer"
```

`inner()` was written inside `outer()`, so its scope chain includes `outer()`.

This is the foundation of **Closures**.

---

# 7. Global Scope

**Global Scope** is the outermost scope of a JavaScript program.

```js
const x = 10;

function test() {
  console.log(x);
}

test(); // 10
```

`test()` can access `x` because it doesn't have its own `x`.

Conceptually:

```text
Global Scope
├── x → 10
└── test → function
```

But the reverse isn't true:

```js
function test() {
  const y = 20;
}

test();

console.log(y); // ReferenceError
```

Global code cannot access a local function variable.

---

# 8. Function Scope

A variable has **function scope** when it is accessible throughout the function where it is declared.

`var` is function-scoped.

```js
function test() {
  var x = 10;

  if (true) {
    var y = 20;
  }

  console.log(x); // 10
  console.log(y); // 20
}
```

Even though `y` is declared inside the `if`, it is accessible throughout the function because `var` is not block-scoped.

### Important

```text
var → Function Scope
```

---

# 9. Block Scope

A variable has **block scope** when it is accessible only inside the `{ }` block where it is declared.

`let` and `const` are block-scoped.

```js
if (true) {
  let x = 10;
  const y = 20;

  console.log(x); // 10
  console.log(y); // 20
}

console.log(x); // ReferenceError
console.log(y); // ReferenceError
```

Conceptually:

```text
Global Scope
└── if Block Scope
    ├── x → 10
    └── y → 20
```

Blocks can be created by:

```js
if (true) {
  let x = 10;
}

for (let i = 0; i < 3; i++) {
  // i is block scoped
}

{
  let y = 20;
}
```

---

# 10. `var` vs `let` vs `const`

| Feature                     | `var`       | `let` | `const` |
| --------------------------- | ----------- | ----- | ------- |
| Scope                       | Function    | Block | Block   |
| Can redeclare in same scope | Yes         | No    | No      |
| Can reassign                | Yes         | Yes   | No      |
| Hoisted                     | Yes         | Yes   | Yes     |
| Access before declaration   | `undefined` | TDZ   | TDZ     |

Example:

```js
function test() {
  var a = 10;

  if (true) {
    let b = 20;
    const c = 30;
  }

  console.log(a); // 10
  console.log(b); // ReferenceError
  console.log(c); // ReferenceError
}
```

---

# 11. Scope Chain Example

```js
const x = 10;

function outer() {
  const y = 20;

  function inner() {
    const z = 30;

    console.log(x);
    console.log(y);
    console.log(z);
  }

  inner();
}

outer();
```

Output:

```text
10
20
30
```

Conceptually:

```text
Global Lexical Environment
└── x = 10
    │
    └── outer() Environment
        └── y = 20
            │
            └── inner() Environment
                └── z = 30
```

Lookup:

```text
inner → outer → global
```

---

# 12. Interview Answers

### What is Lexical Environment?

> "A Lexical Environment is an internal structure used by JavaScript to store variable and function bindings and maintain a relationship with the outer environment. It helps JavaScript resolve variables through the scope chain."

### What is Scope Chain?

> "The Scope Chain is the chain of lexical environments JavaScript searches when resolving a variable. It starts from the current scope and moves outward until the variable is found or the global scope is reached."

### What is Global Scope?

> "Global Scope is the outermost scope of a JavaScript program. Variables declared there can generally be accessed from nested scopes unless they are shadowed."

### What is Function Scope?

> "Function Scope means a variable is accessible throughout the function in which it is declared. `var` is function-scoped."

### What is Block Scope?

> "Block Scope means a variable is accessible only inside the block where it is declared. `let` and `const` are block-scoped."

---

# Key Takeaways

- **Execution Context** → overall environment in which code executes.
- **Lexical Environment** → manages variable/function bindings and scope relationships.
- **Variable Environment** → specification concept related to variable declarations.
- **Scope Chain** → path JavaScript follows to resolve variables.
- Variable lookup goes **inner → outer → global**.
- JavaScript uses **lexical scoping**.
- Inner variables can **shadow** outer variables.
- `var` → **function scoped**.
- `let` → **block scoped**.
- `const` → **block scoped**.
- `let` and `const` have a **Temporal Dead Zone (TDZ)** before initialization.
- Function scope and block scope are different.
- Lexical Environment + Scope Chain are fundamental to understanding **Closures**.
