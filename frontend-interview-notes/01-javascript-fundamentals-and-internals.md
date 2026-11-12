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

# JavaScript — Variables & Hoisting

## 1. `var` vs `let` vs `const`

| Feature                        | `var`       | `let`  | `const` |
| ------------------------------ | ----------- | ------ | ------- |
| Function scoped                | ✅          | ❌     | ❌      |
| Block scoped                   | ❌          | ✅     | ✅      |
| Redeclaration                  | ✅          | ❌     | ❌      |
| Reassignment                   | ✅          | ✅     | ❌      |
| Hoisted/processed              | ✅          | ✅     | ✅      |
| Initialized before declaration | `undefined` | ❌ TDZ | ❌ TDZ  |

### `var`

```js
function test() {
  if (true) {
    var x = 10;
  }

  console.log(x); // 10
}
```

`var` is function-scoped, not block-scoped.

### `let`

```js
if (true) {
  let x = 10;
}

console.log(x); // ReferenceError
```

`let` is block-scoped.

### `const`

```js
const x = 10;

x = 20; // TypeError
```

`const` cannot be reassigned.

But objects declared with `const` can still be mutated:

```js
const user = {
  name: "Vipin",
};

user.name = "Rahul"; // ✅
```

`const` protects the binding, not the object's contents.

---

# 2. Hoisting

Hoisting is the behavior where JavaScript processes declarations before executing the code in that scope.

A better technical explanation:

> JavaScript creates bindings during the creation/setup phase before execution reaches the declaration.

It does **not literally move code to the top**.

---

## `var` Hoisting

```js
console.log(x);

var x = 10;
```

Output:

```text
undefined
```

Mental model:

```text
Creation:
x → undefined

Execution:
console.log(x) → undefined
x = 10
```

### Interview answer

> `var` is hoisted and initialized with `undefined` during the creation phase.

---

# 3. Temporal Dead Zone (TDZ)

`let` and `const` are hoisted/processed, but their bindings remain **uninitialized** until execution reaches their declaration.

The period between entering the scope and initialization is the **Temporal Dead Zone**.

```js
console.log(x); // ReferenceError

let x = 10;
```

Mental model:

```text
Scope entered
     ↓
x → uninitialized
     ↓
    TDZ
     ↓
console.log(x)
     ↓
ReferenceError
     ↓
let x = 10
     ↓
x → 10
```

### Important

Do not say:

> "`let` and `const` are not hoisted."

Better:

> `let` and `const` are hoisted/processed, but remain uninitialized in the TDZ.

---

## TDZ ends at initialization

```js
console.log(a); // ReferenceError

let a;

console.log(a); // undefined
```

After execution reaches:

```js
let a;
```

`a` is initialized with `undefined`.

```text
Before `let a`:
a → uninitialized → TDZ

After `let a`:
a → undefined
```

---

# 4. Variable Initialization

Always distinguish:

```text
Declaration → Initialization → Assignment
```

Example:

```js
var x = 10;
```

Conceptually:

```text
Declaration:
x exists

Initialization:
x = undefined

Assignment:
x = 10
```

For `let`:

```js
let x = 10;
```

```text
Creation:
x → uninitialized
     ↓
    TDZ

Execution reaches declaration:
x → 10
```

For:

```js
let x;
```

After execution reaches the declaration:

```text
x → undefined
```

### `const` must be initialized

```js
const x; // SyntaxError
```

Unlike `let`:

```js
let x; // ✅
```

---

# 5. Function Hoisting

Function declarations are hoisted with their function object.

```js
sayHello();

function sayHello() {
  console.log("Hello");
}
```

Output:

```text
Hello
```

Mental model:

```text
Creation phase:

sayHello → actual function
```

Therefore the function can be called before its declaration.

### Interview answer

> Function declarations are hoisted with their function object, so they can be called before their declaration in the source code.

---

# 6. Function Declaration vs Function Expression

## Function Declaration

```js
greet();

function greet() {
  console.log("Hello");
}
```

✅ Works.

---

## Function Expression with `var`

```js
greet();

var greet = function () {
  console.log("Hello");
};
```

❌ `TypeError`

Because:

```text
Creation:
greet → undefined

Execution:
greet()
↓
undefined()
↓
TypeError
```

---

## Function Expression with `let`

```js
greet();

let greet = function () {
  console.log("Hello");
};
```

❌ `ReferenceError`

Because `greet` is in the TDZ.

---

## Function Expression with `const`

```js
greet();

const greet = function () {
  console.log("Hello");
};
```

❌ `ReferenceError`

Same TDZ behavior.

---

# 7. `typeof` + Hoisting Trap

With `var`:

```js
console.log(typeof greet);

var greet = function () {};
```

Output:

```text
"undefined"
```

But with `let`:

```js
console.log(typeof greet);

let greet = function () {};
```

Output:

```text
ReferenceError
```

Why?

`typeof` does not bypass the TDZ.

Compare:

```js
console.log(typeof unknownVariable);
```

Output:

```text
"undefined"
```

A completely undeclared identifier is different from a declared-but-uninitialized `let`/`const` binding.

---

# 8. Interview Trap — Function Declaration + `var`

```js
sayHello();

var sayHello = function () {
  console.log("Hello");
};

function sayHello() {
  console.log("Hi");
}
```

Output:

```text
Hi
```

### Why?

During creation:

```text
sayHello → function declaration
```

The function declaration is available.

During execution:

```js
sayHello();
```

calls the `"Hi"` function.

Then:

```js
var sayHello = function () {
  console.log("Hello");
};
```

reassigns the existing binding.

Now:

```text
sayHello → "Hello" function
```

Therefore:

```js
sayHello();
```

after the assignment would print:

```text
Hello
```

Complete example:

```js
sayHello();

var sayHello = function () {
  console.log("Hello");
};

function sayHello() {
  console.log("Hi");
}

sayHello();
```

Output:

```text
Hi
Hello
```

---

# 9. Interview Trap — Function Declaration + `var` Assignment

```js
foo();

var foo = 10;

function foo() {
  console.log("Hello");
}

foo();
```

Output:

```text
Hello
TypeError
```

### Why?

Creation phase:

```text
foo → function
```

First call:

```text
foo() → Hello
```

Then:

```js
foo = 10;
```

Now:

```text
foo → 10
```

Second call:

```js
foo();
```

is effectively:

```js
10();
```

Therefore:

```text
TypeError
```

---

# 10. Interview Trap — `var` Function Expression

```js
foo();

var foo = function () {
  console.log("Hello");
};
```

Output:

```text
TypeError
```

Because:

```text
Creation:
foo → undefined

Execution:
foo()
↓
undefined()
↓
TypeError
```

---

# 11. Interview Trap — `let` Function Expression

```js
foo();

let foo = function () {
  console.log("Hello");
};
```

Output:

```text
ReferenceError
```

Because:

```text
foo → uninitialized
     ↓
    TDZ
     ↓
foo()
     ↓
ReferenceError
```

---

# 12. Quick Comparison

```text
Function Declaration
─────────────────────
function foo() {}
        ↓
foo → actual function
        ↓
can call before declaration
```

```text
var Function Expression
────────────────────────
var foo = function() {}
        ↓
foo → undefined
        ↓
calling before assignment
        ↓
TypeError
```

```text
let/const Function Expression
──────────────────────────────
let/const foo = function() {}
        ↓
foo → uninitialized
        ↓
TDZ
        ↓
access before declaration
        ↓
ReferenceError
```

---

# 13. Common Interview Traps

### Trap 1

```js
console.log(a);

var a = 10;
```

Output:

```text
undefined
```

---

### Trap 2

```js
console.log(a);

let a = 10;
```

Output:

```text
ReferenceError
```

---

### Trap 3

```js
foo();

var foo = function () {};
```

Output:

```text
TypeError
```

---

### Trap 4

```js
foo();

let foo = function () {};
```

Output:

```text
ReferenceError
```

---

### Trap 5

```js
foo();

function foo() {}
```

Works because function declarations are hoisted with the function object.

---

# 14. Interview-Ready Answers

### What is hoisting?

> Hoisting is the behavior where JavaScript creates/initializes certain bindings during the creation phase before executing the code in that scope. It is not literally moving code to the top.

### Is `let` hoisted?

> Yes, its binding is created during the creation phase, but it remains uninitialized in the Temporal Dead Zone until execution reaches the declaration.

### Why does `var` return `undefined` before declaration?

> Because the `var` binding is initialized with `undefined` during the creation phase.

### Why does `let` throw ReferenceError before declaration?

> Because the binding exists but remains uninitialized in the Temporal Dead Zone.

### Why can function declarations be called before declaration?

> Because function declarations are instantiated with their function object during the creation phase.

### Why does a function expression with `var` give TypeError?

> Because the `var` binding initially contains `undefined`, and calling `undefined` as a function causes a TypeError.

---

# ⭐ Key Takeaways

```text
1. var → function scoped
2. let/const → block scoped

3. var → hoisted + initialized as undefined
4. let/const → hoisted + uninitialized → TDZ

5. TDZ ends when execution reaches the declaration

6. let x; → x becomes undefined after initialization

7. const must be initialized at declaration

8. Function declarations are hoisted with their function object

9. var function expression before assignment → TypeError

10. let/const function expression before declaration → ReferenceError

11. typeof does NOT bypass TDZ

12. Hoisting does not literally mean moving code to the top
```

This completes your **Variables & Hoisting** bucket. Next in the roadmap, we can move into the next JavaScript internals topic while keeping the same **concept → output questions → interview traps → GitHub notes** approach.

# JavaScript — Primitive vs Reference Types & Type Coercion

> Interview-focused notes with mental models, examples, common traps, and key takeaways.

---

# 1. Primitive vs Reference Types

## Primitive Types

JavaScript has **7 primitive types**:

```text
string
number
boolean
undefined
null
bigint
symbol
```

Primitives represent **single immutable values**.

```js
let x = 10;
let y = x;

y = 20;

console.log(x); // 10
console.log(y); // 20
```

### Mental Model

When assigning a primitive:

```text
x ──→ 10

y = x

x ──→ 10
y ──→ 10

y = 20

x ──→ 10
y ──→ 20
```

The value is copied.

---

# Reference Types

Objects, arrays, and functions are reference values.

```js
const user1 = {
  name: "Vipin",
};

const user2 = user1;

user2.name = "Rahul";

console.log(user1.name); // Rahul
console.log(user2.name); // Rahul
```

### Mental Model

```text
user1 ──────┐
            ↓
       ┌──────────────┐
       │ name: Rahul  │
       └──────────────┘
            ↑
user2 ──────┘
```

Both variables refer to the **same object**.

### Important Interview Wording

Don't say:

> "user1 contains the object."

Better:

> "`user1` holds a reference to the object."

---

# Primitive vs Reference

| Primitive                   | Reference                               |
| --------------------------- | --------------------------------------- |
| Stores a value              | Variable holds a reference to an object |
| Immutable                   | Objects are generally mutable           |
| Assignment copies the value | Assignment copies the reference         |
| Compared by value           | Objects compared by reference identity  |

Example:

```js
let a = 10;
let b = a;

b = 20;

console.log(a); // 10
```

Versus:

```js
const a = { value: 10 };
const b = a;

b.value = 20;

console.log(a.value); // 20
```

---

# Object Equality

```js
const a = {};
const b = {};
const c = a;

console.log(a === b); // false
console.log(a === c); // true
```

Why?

```text
a ─────→ Object #1
c ─────→ Object #1

b ─────→ Object #2
```

`a` and `b` are different objects.

`a` and `c` reference the same object.

### Key Rule

```js
{} === {} // false
```

Two separately created objects are different references.

```js
const a = {};
const b = a;

a === b; // true
```

---

# Shallow Copy

The spread operator creates a **shallow copy**:

```js
const user1 = {
  name: "Vipin",
  address: {
    city: "Noida",
  },
};

const user2 = { ...user1 };

user2.name = "Rahul";
user2.address.city = "Delhi";

console.log(user1.name); // "Vipin"
console.log(user1.address.city); // "Delhi"
```

### Why?

Top-level properties are copied:

```text
user1 ─────→ Object #1
             name: "Vipin"
             address ─────┐
                          │
user2 ─────→ Object #2    │
             name: "Rahul"│
             address ─────┘
                          ↓
                    { city: "Delhi" }
```

`name` is a primitive → copied independently.

`address` is an object → nested reference is shared.

### Interview Answer

> A shallow copy creates a new top-level object, but nested objects and arrays still contain references to the original nested values.

---

# Important Memory Note

Avoid saying:

> "Primitives are always stored in Stack and objects are always stored in Heap."

That's an oversimplification.

The exact memory implementation depends on the JavaScript engine.

For interviews, focus on the observable behavior:

```text
Primitive assignment → value copied

Object assignment → reference copied
```

---

# Primitive vs Reference — Interview Traps

### Trap 1

```js
let a = 10;
let b = a;

b = 20;

console.log(a); // 10
console.log(b); // 20
```

### Trap 2

```js
const a = { value: 10 };
const b = a;

b.value = 20;

console.log(a.value); // 20
```

### Trap 3

```js
const a = {};
const b = {};

console.log(a === b); // false
```

### Trap 4

```js
const a = {};
const b = a;

console.log(a === b); // true
```

### Trap 5 — Shallow Copy

```js
const a = {
  name: "Vipin",
  address: {
    city: "Noida",
  },
};

const b = { ...a };

b.name = "Rahul";
b.address.city = "Delhi";

console.log(a.name); // "Vipin"
console.log(a.address.city); // "Delhi"
```

---

# Key Takeaways — Primitive vs Reference

- JavaScript has **7 primitive types**.
- Primitive values are immutable.
- Primitive assignment copies the value.
- Objects/arrays/functions are reference values.
- Object assignment copies the reference.
- Two variables can reference the same object.
- `===` compares object identity/reference.
- `{}` and `{}` are different objects.
- `{ ...obj }` creates a shallow copy.
- Shallow copies don't recursively copy nested objects.
- Avoid oversimplifying JavaScript memory as "stack vs heap."

---

# 2. ⭐ Type Coercion

## What is Type Coercion?

**Type coercion** is the conversion of one type into another during an operation.

### Implicit Coercion

JavaScript performs the conversion automatically:

```js
"5" - 2; // 3
```

### Explicit Coercion

The developer performs the conversion:

```js
Number("5"); // 5
String(5); // "5"
Boolean(1); // true
```

---

# The Type Coercion Mental Model

Whenever you see a coercion question, ask:

```text
1. Which operator is being used?
          ↓
2. What does that operator expect?
          ↓
3. How are the values converted?
          ↓
4. What is the final operation?
```

This is much better than memorizing random outputs.

---

# `+` Is Special

The `+` operator can perform:

```text
Number + Number → addition

String + anything → string concatenation
```

Example:

```js
"5" + 2;
```

Because a string is involved:

```text
"5" + 2
 ↓
"5" + "2"
 ↓
"52"
```

```js
"5" + 2; // "52"
"5" + true; // "5true"
"5" + null; // "5null"
```

### Important Trap

Do NOT think:

```text
"5" + true
↓
"5" + 1
↓
6
```

That's wrong.

`+` chooses string concatenation when a string is involved.

---

# Numeric Operators

These generally perform numeric coercion:

```text
-
*
/
%
```

Example:

```js
"5" - 2; // 3
```

Conversion:

```text
"5" → 5

5 - 2 → 3
```

Examples:

```js
"5" - 2; // 3
"5" * 2; // 10
"5" / 2; // 2.5
"5" % 2; // 1
```

---

# Number Conversion Table

When JavaScript needs a number:

| Value       | `Number(value)` |
| ----------- | --------------: |
| `"5"`       |             `5` |
| `""`        |             `0` |
| `" "`       |             `0` |
| `"hello"`   |           `NaN` |
| `true`      |             `1` |
| `false`     |             `0` |
| `null`      |             `0` |
| `undefined` |           `NaN` |
| `[]`        |             `0` |
| `[5]`       |             `5` |
| `[1, 2]`    |           `NaN` |
| `{}`        |           `NaN` |

---

# Boolean Numeric Coercion

When numeric conversion happens:

```text
true  → 1
false → 0
```

Therefore:

```js
true + 1; // 2
false + 1; // 1
```

But remember:

```js
typeof true; // "boolean"
```

`true` doesn't become a number permanently.

It is only converted to a number **for that operation**.

---

# `null` vs `undefined`

This is a very common interview topic.

### Numeric conversion

```text
null      → 0
undefined → NaN
```

Therefore:

```js
null + 1; // 1
undefined + 1; // NaN

null * 5; // 0
undefined * 5; // NaN
```

### But don't generalize this to equality

```js
null == 0; // false
null === 0; // false
```

`==` has its own special rules.

---

# Empty String

```js
"" + 1;
```

Because `+` performs string concatenation:

```text
"" + 1
↓
"" + "1"
↓
"1"
```

Result:

```js
"1";
```

But:

```js
"" - 1;
```

`-` forces numeric conversion:

```text
"" → 0
0 - 1 → -1
```

Result:

```js
-1;
```

---

# Object / Array Coercion

Objects can be converted to primitive values during certain operations.

Some important conversions:

```text
[]    → ""
[5]   → "5"
{}    → "[object Object]"
```

Therefore:

```js
[] + [];
```

Conceptually:

```text
[] → ""
[] → ""

"" + ""
↓
""
```

Result:

```js
"";
```

---

# `[] + {}`

```js
[] + {};
```

Conceptually:

```text
[] → ""
{} → "[object Object]"

"" + "[object Object]"
```

Result:

```js
"[object Object]";
```

---

# `{}` + `[]` — Famous Parsing Trap

This is tricky because JavaScript's parser matters.

```js
{
}
+[];
```

At the beginning of a statement, `{}` can be interpreted as an empty block rather than an object literal.

But:

```js
({}) + [];
```

forces `{}` to be treated as an object expression.

Result:

```js
({}) + []; // "[object Object]"
```

### Interview Lesson

Don't blindly memorize the output of:

```js
{
}
+[];
```

The parsing context matters.

---

# Famous Type Coercion Interview Questions

## 1. String + Number

```js
"10" + 5; // "105"
```

## 2. String - Number

```js
"10" - 5; // 5
```

## 3. Boolean + Number

```js
true + 1; // 2
false + 1; // 1
```

## 4. String + Boolean

```js
"5" + true; // "5true"
"5" + false; // "5false"
```

## 5. String - Boolean

```js
"5" - true; // 4
"5" - false; // 5
```

## 6. `null`

```js
null + 1; // 1
null * 5; // 0
```

## 7. `undefined`

```js
undefined + 1; // NaN
undefined * 5; // NaN
```

## 8. String + null

```js
"5" + null; // "5null"
```

## 9. String + undefined

```js
"5" + undefined; // "5undefined"
```

## 10. Empty String

```js
"" + 1; // "1"
"" - 1; // -1
```

## 11. Arrays

```js
[] + []; // ""
[] + {}; // "[object Object]"
```

## 12. Equality Preview

```js
null == 0; // false
null === 0; // false
```

Equality rules will be covered separately.

---

# High-Value Interview Traps

### Trap A

```js
console.log("5" + true);
```

Output:

```text
"5true"
```

Not `6`.

---

### Trap B

```js
console.log("5" - true);
```

Output:

```text
4
```

Because:

```text
"5" → 5
true → 1

5 - 1 → 4
```

---

### Trap C

```js
console.log(null + 1);
console.log(undefined + 1);
```

Output:

```text
1
NaN
```

---

### Trap D

```js
console.log([] + []);
console.log([] + {});
```

Output:

```text
""
"[object Object]"
```

---

# Type Coercion vs Type Conversion

These terms are often used interchangeably, but for interviews:

### Type Coercion

Automatic:

```js
"5" - 2; // 3
```

### Type Conversion

Explicit:

```js
Number("5"); // 5
```

A useful distinction:

```text
Coercion   → usually implicit
Conversion → usually explicit
```

---

# Interview-Ready Answer

### What is type coercion?

> Type coercion is JavaScript's process of converting a value from one type to another during an operation. It can happen implicitly, such as `"5" - 2`, where `"5"` is converted to `5`, or explicitly using functions like `Number()`, `String()`, and `Boolean()`.

### Why is `+` different?

> The `+` operator can perform either numeric addition or string concatenation. If a string is involved, JavaScript may convert the other operand to a string and concatenate.

---

# Key Takeaways

- Type coercion = conversion between types during an operation.
- Implicit coercion happens automatically.
- Explicit conversion is performed by the developer.
- `+` is special.
- `+` can perform addition or string concatenation.
- `-`, `*`, `/`, `%` generally force numeric conversion.
- `true → 1` during numeric conversion.
- `false → 0` during numeric conversion.
- `null → 0` during numeric conversion.
- `undefined → NaN` during numeric conversion.
- `NaN` is the result of an invalid numeric conversion.
- Objects/arrays can be converted to primitive values.
- Don't memorize random coercion outputs; identify the operator and derive the conversion.
- `==` has special coercion rules; `===` compares without type coercion.

# JavaScript: Equality, Values & Operators

Interview-focused notes covering:

- `==` vs `===`
- Truthy / Falsy
- `null` vs `undefined`
- `NaN`
- `typeof`
- `instanceof` _(to revisit with Prototypes)_
- Optional Chaining `?.`
- Nullish Coalescing `??`

---

## 1. `==` vs `===`

### `==` — Loose Equality

Allows **type coercion** before comparison.

```js
5 == "5"; // true
0 == false; // true
```

### `===` — Strict Equality

Checks both **value and type** without coercion.

```js
5 === "5"; // false
0 === false; // false
```

### Mental Model

```text
==

Different types?
     ↓
Type coercion may happen
     ↓
Compare


===

Different types?
     ↓
YES → false
NO  → compare values
```

### Important Trap

```js
null == undefined; // true
null === undefined; // false

null == 0; // false
null == false; // false
```

`null` and `undefined` have a special relationship under `==`.

### Interview Rule

> Prefer `===` in normal application code because it avoids unexpected type coercion.

---

# 2. Truthy / Falsy

JavaScript converts values to Boolean when they are used in a Boolean context.

```js
if (value) {
  // ...
}
```

### Falsy Values

The main falsy values are:

```js
false;
0 - 0;
0n;
("");
null;
undefined;
NaN;
```

Everything else is generally **truthy**.

### Important Traps

```js
Boolean("0"); // true
Boolean("false"); // true

Boolean([]); // true
Boolean({}); // true
```

An empty string is falsy, but an empty array/object is truthy.

### Mental Model

```text
Value
  ↓
Is it one of JavaScript's falsy values?
  ↓
YES → false
NO  → true
```

### Interview Trap

```js
if ([]) {
  console.log("Yes");
}
```

Output:

```text
Yes
```

Because arrays are objects, and objects are truthy.

---

# 3. `null` vs `undefined`

### `undefined`

Usually means:

> A value hasn't been assigned/provided.

```js
let user;

console.log(user); // undefined
```

### `null`

Usually means:

> The developer intentionally represents "no value".

```js
let user = null;
```

### Mental Model

```text
undefined
    ↓
Value is missing / not assigned


null
    ↓
Value is intentionally empty
```

### Equality

```js
null == undefined; // true
null === undefined; // false
```

### Famous Trap

```js
typeof null; // "object"
```

This is a historical JavaScript behavior.

`null` conceptually represents an absence of value; `typeof` reporting `"object"` is legacy behavior.

### Interview-ready Explanation

> "`undefined` generally represents a missing or unassigned value, while `null` is an explicitly assigned empty value. `null == undefined` is true because of a special loose-equality rule, but strict equality returns false."

---

# 4. `NaN`

`NaN` means **Not-a-Number**.

It represents an invalid numeric result.

```js
"hello" * 2; // NaN
0 / 0; // NaN
```

### Famous Trap

```js
typeof NaN; // "number"
```

`NaN` belongs to JavaScript's `Number` type.

### Another Trap

```js
NaN === NaN; // false
```

`NaN` is not equal to itself using normal equality.

### Correct Check

Prefer:

```js
Number.isNaN(value);
```

### `Number.isNaN()` vs `isNaN()`

```js
Number.isNaN("hello");
// false
```

It does **not** perform type coercion.

Global `isNaN()` performs conversion:

```js
isNaN("hello");
// true
```

Because:

```text
"hello"
   ↓
Number("hello")
   ↓
NaN
   ↓
isNaN → true
```

But:

```js
isNaN("123");
// false
```

because `"123"` converts to `123`.

### Interview Rule

> Use `Number.isNaN()` when you specifically want to detect the actual `NaN` value.

---

# 5. `typeof`

`typeof` tells you the runtime type category of a value.

```js
typeof "hello"; // "string"
typeof 42; // "number"
typeof true; // "boolean"
typeof undefined; // "undefined"
typeof null; // "object"
typeof {}; // "object"
typeof []; // "object"
typeof function () {}; // "function"
typeof Symbol(); // "symbol"
typeof 10n; // "bigint"
```

### Famous Traps

```js
typeof null; // "object"
typeof []; // "object"
```

### Arrays

`typeof` cannot specifically identify arrays.

Use:

```js
Array.isArray([]);
```

Result:

```js
true;
```

### Mental Model

```text
typeof value
     ↓
Runtime type category
```

It doesn't give you a complete description of an object's structure.

---

# 6. `instanceof`

We only introduced this topic and intentionally postponed detailed practice until learning **Prototypes and the Prototype Chain**.

Core idea:

> `instanceof` checks whether a constructor's `.prototype` exists somewhere in an object's prototype chain.

Example:

```js
const arr = [];

arr instanceof Array; // true
arr instanceof Object; // true
```

Mental model:

```text
arr
 ↓
Array.prototype
 ↓
Object.prototype
 ↓
null
```

When we cover:

```text
Objects
 ↓
Prototype
 ↓
Prototype Chain
 ↓
Constructor Functions
 ↓
instanceof
 ↓
Classes
```

we will revisit this properly with output questions and interview traps.

---

# 7. Optional Chaining `?.`

Optional chaining safely accesses properties/methods when a value may be `null` or `undefined`.

Without it:

```js
const user = {};

user.profile.name;
// TypeError
```

With it:

```js
user.profile?.name;
// undefined
```

### Mental Model

```text
user.profile?.name
       ↓
Is profile null/undefined?
       ↓
YES → undefined
NO  → access name
```

### Nested Optional Chaining

```js
user?.profile?.address?.city;
```

Each `?.` protects that particular chain segment.

### Function Calls

```js
const user = {};

user.login?.();
// undefined
```

Without `?.`:

```js
user.login();
// TypeError
```

### Important Trap

This:

```js
user.profile?.address.city;
```

does **NOT** fully protect the chain.

If `address` is `undefined`, JavaScript eventually evaluates:

```js
undefined.city;
```

→ `TypeError`

Use:

```js
user.profile?.address?.city;
```

if `address` may also be missing.

### Interview Rule

> Optional chaining prevents errors when the value immediately before `?.` is `null` or `undefined`, returning `undefined` instead.

---

# 8. Nullish Coalescing `??`

`??` provides a fallback when the left side is specifically:

```js
null;
```

or

```js
undefined;
```

Example:

```js
const name = null ?? "Guest";

console.log(name);
// "Guest"
```

### Mental Model

```text
value
 ↓
null or undefined?
 ↓
YES → use fallback
NO  → use original value
```

### Important Difference: `??` vs `||`

`||` checks **truthiness**.

`??` checks only **nullish values**:

```text
null
undefined
```

Therefore:

```js
0 || 100;
// 100

0 ?? 100;
// 0
```

More examples:

```js
"" || "Guest"; // "Guest"
"" ?? "Guest"; // ""

false || true; // true
false ?? true; // false

null || "Guest"; // "Guest"
null ?? "Guest"; // "Guest"

undefined || 10; // 10
undefined ?? 10; // 10
```

### Practical Use

Suppose `0` is a valid value:

```js
const quantity = 0;

const result = quantity ?? 10;

console.log(result);
// 0
```

Using `||` would incorrectly replace it:

```js
const result = quantity || 10;

// 10
```

### Interview Rule

> Use `??` when you want a fallback only for `null` or `undefined`. Use `||` when you intentionally want a fallback for any falsy value.

---

# Quick Interview Cheat Sheet

| Concept             | Key Rule                                                   |     |                           |
| ------------------- | ---------------------------------------------------------- | --- | ------------------------- |
| `==`                | Allows type coercion                                       |     |                           |
| `===`               | Compares type + value without coercion                     |     |                           |
| Truthy              | Values that become `true` in Boolean context               |     |                           |
| Falsy               | `false`, `0`, `-0`, `0n`, `""`, `null`, `undefined`, `NaN` |     |                           |
| `undefined`         | Generally missing/unassigned value                         |     |                           |
| `null`              | Explicitly represents absence of value                     |     |                           |
| `NaN`               | Invalid numeric result                                     |     |                           |
| `typeof NaN`        | `"number"`                                                 |     |                           |
| `typeof null`       | `"object"` — historical behavior                           |     |                           |
| `typeof []`         | `"object"`                                                 |     |                           |
| `Array.isArray([])` | `true`                                                     |     |                           |
| `instanceof`        | Checks prototype-chain relationship                        |     |                           |
| `?.`                | Safely accesses/calls when value may be nullish            |     |                           |
| `??`                | Fallback only for `null` / `undefined`                     |     |                           |
| `                   |                                                            | `   | Fallback for falsy values |

---

# Key Interview Traps

```js
typeof null;
// "object"
```

```js
typeof NaN;
// "number"
```

```js
NaN === NaN;
// false
```

```js
Boolean([]);
// true
```

```js
Boolean({});
// true
```

```js
null == undefined;
// true
```

```js
null === undefined;
// false
```

```js
0 || 100;
// 100
```

```js
0 ?? 100;
// 0
```

```js
"" || "Guest";
// "Guest"
```

```js
"" ?? "Guest";
// ""
```

```js
user.profile?.address.city;
// Can still throw if address is undefined
```

```js
user.profile?.address?.city;
// Safely returns undefined
```

---

# Core Mental Models

## Equality

```text
==  → "Can coercion make these equal?"
=== → "Are type AND value already equal?"
```

## Truthiness

```text
Value
 ↓
Falsy value?
 ↓
YES → false
NO  → true
```

## Optional Chaining

```text
value?.property
       ↓
null / undefined?
       ↓
YES → undefined
NO  → continue
```

## Nullish Coalescing

```text
value ?? fallback
       ↓
null / undefined?
       ↓
YES → fallback
NO  → value
```

---

# Final Takeaways

1. Prefer `===` over `==` unless you intentionally need loose equality.
2. Learn the falsy values instead of assuming "empty" means falsy.
3. `null` and `undefined` are different concepts.
4. `NaN` is a `number` and is not equal to itself.
5. `Number.isNaN()` is generally safer for detecting `NaN`.
6. `typeof` has historical traps, especially `typeof null`.
7. `instanceof` becomes much easier after understanding prototypes.
8. `?.` prevents errors when accessing potentially nullish values.
9. `??` is different from `||`: it only treats `null` and `undefined` as missing.
10. For interview questions, derive the result from the underlying rule instead of memorizing outputs.

# JavaScript Strict Mode

Strict Mode makes JavaScript enforce stricter rules and helps catch certain programming mistakes that non-strict/sloppy mode may silently allow.

Enable it with:

```js
"use strict";
```

It can be applied to an entire script or to a specific function.

---

## 1. Why Strict Mode?

Without strict mode, some problematic operations may be silently accepted.

With strict mode, JavaScript often throws an error instead.

### Mental Model

```text
Non-strict / Sloppy Mode
        ↓
Some problematic operations
        ↓
JavaScript may tolerate them


Strict Mode
        ↓
Same operation
        ↓
JavaScript throws an error
```

The goal is to catch mistakes earlier and make JavaScript behavior more predictable.

---

# 2. Enabling Strict Mode

### Entire script

```js
"use strict";

const name = "Vipin";
```

Everything in that script runs in strict mode.

### Function-level

```js
function test() {
  "use strict";

  // strict mode only inside this function
}
```

---

# 3. Accidental Global Variables

One of the most common strict-mode examples:

### Non-strict

```js
x = 10;

console.log(x);
```

In classic non-strict script environments, this can create/assign a global variable.

### Strict mode

```js
"use strict";

x = 10;
```

Throws:

```text
ReferenceError
```

### Why?

You attempted to assign a value to an undeclared variable.

Strict mode forces you to explicitly declare variables:

```js
"use strict";

let x = 10;
```

---

# 4. Assigning to Read-Only Properties

Consider:

```js
const obj = {};

Object.defineProperty(obj, "name", {
  value: "Vipin",
  writable: false,
});
```

Now `name` cannot be changed.

### Strict mode

```js
"use strict";

obj.name = "John";
```

Throws:

```text
TypeError
```

In non-strict mode, this kind of invalid assignment can fail silently.

### Mental Model

```text
Non-strict
invalid assignment
      ↓
may fail silently


Strict
invalid assignment
      ↓
TypeError
```

---

# 5. Deleting Variables

Strict mode does not allow deleting declared variables.

```js
"use strict";

let x = 10;

delete x;
```

This results in a:

```text
SyntaxError
```

---

# 6. Strict Mode and `this`

This is an important strict-mode behavior, but the complete `this` topic will be covered separately.

For a **regular function called without an object**, strict mode gives:

```js
"use strict";

function test() {
  console.log(this);
}

test();
```

Output:

```text
undefined
```

### Non-strict

In a classic browser script, a plain function call traditionally gets the global object:

```js
function test() {
  console.log(this);
}

test();

// window
```

### Mental Model

```text
test()
 ↓
Plain function call
 ↓
Strict?
 ├── YES → this = undefined
 └── NO  → global object in classic non-strict scripts
```

---

# 7. Strict Mode Does NOT Mean `this` Is Always `undefined`

This is an important distinction.

```js
"use strict";

const user = {
  name: "Vipin",

  greet() {
    console.log(this.name);
  },
};

user.greet();
```

Output:

```text
Vipin
```

Why?

Because:

```js
user.greet();
```

is a **method call**.

The object before `.` becomes the `this` value:

```text
user.greet()
    ↑
    this
```

So:

```js
this === user;
```

is `true`.

---

# 8. Detached Method Trap

Consider:

```js
"use strict";

const user = {
  name: "Vipin",

  greet() {
    console.log(this.name);
  },
};

user.greet();
```

Output:

```text
Vipin
```

But:

```js
const fn = user.greet;

fn();
```

is different.

Now it is a plain function call:

```text
user.greet()
     ↓
     this = user


fn()
 ↓
plain function call
 ↓
this = undefined
```

Therefore:

```js
this.name;
```

attempts to access `.name` on `undefined`, causing:

```text
TypeError
```

### Interview takeaway

> The function itself did not change. The way the function was called changed.

The complete behavior of `this`, including arrow functions, `call`, `apply`, `bind`, constructors, classes, and event handlers, should be studied separately.

---

# 9. Strict vs Non-Strict — Quick Comparison

| Behavior                       | Strict Mode      | Non-Strict Mode                  |
| ------------------------------ | ---------------- | -------------------------------- |
| Undeclared variable assignment | `ReferenceError` | May create/assign global         |
| Invalid read-only assignment   | `TypeError`      | May fail silently                |
| Delete declared variable       | `SyntaxError`    | Different/sloppy behavior        |
| Plain regular function `this`  | `undefined`      | Global object in classic scripts |
| General error handling         | Stricter         | More permissive                  |

---

# 10. Interview Traps

### Trap 1 — Undeclared variable

```js
"use strict";

x = 10;
```

```text
ReferenceError
```

---

### Trap 2 — `this` in a plain function

```js
"use strict";

function test() {
  console.log(this);
}

test();
```

```text
undefined
```

---

### Trap 3 — Method call

```js
"use strict";

const user = {
  name: "Vipin",

  greet() {
    console.log(this.name);
  },
};

user.greet();
```

```text
Vipin
```

Strict mode doesn't make method-call `this` become `undefined`.

---

### Trap 4 — Detached method

```js
"use strict";

const user = {
  name: "Vipin",

  greet() {
    console.log(this.name);
  },
};

const fn = user.greet;

fn();
```

This throws a `TypeError` because `this` is `undefined`.

---

# 11. Interview-Ready Explanation

> "Strict mode is enabled using `'use strict'` and makes JavaScript enforce stricter rules. It prevents things like accidental global variable creation and causes certain invalid operations to throw errors instead of failing silently. It also changes the behavior of `this` in a plain regular function call: in strict mode, `this` is `undefined` instead of being automatically substituted with the global object. However, `this` in a method call still depends on the call site. I'll treat the complete `this` behavior separately because it also involves arrow functions, call/apply/bind, constructors, and prototypes."

---

# Key Takeaways

```text
"use strict"
     ↓
Stricter JavaScript behavior
```

Remember:

1. Strict mode catches certain programming mistakes.
2. Use `"use strict"` to enable it.
3. Undeclared assignments throw `ReferenceError`.
4. Invalid assignments that may silently fail in sloppy mode can throw errors.
5. Declared variables cannot be deleted.
6. Plain regular function call → `this === undefined` in strict mode.
7. Method call → `this` is still determined by the object used for the call.
8. Detached methods can lose their original `this`.
9. Arrow-function `this` and the complete `this` mechanism will be covered separately.
