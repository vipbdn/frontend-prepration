# 03 — Objects & Prototypes 🔥

> **Interview + Understanding Guide** — `00-starter-roadmap.md` Section 3
>
> Goal: Understand how objects really work in JavaScript — references, the prototype chain, how `new` and `class` work under the hood — and implement `new` and `instanceof` from scratch.

---

# 1. Objects & References

Objects are **reference types**. A variable holds a **reference** (address) to the object in the heap, not the object itself.

```js
const a = { count: 1 };
const b = a; // copies the reference, not the object

b.count = 99;
console.log(a.count); // 99
console.log(a === b); // true — same object
```

```text
Stack            Heap
a ──────┐
        ├──▶ { count: 99 }
b ──────┘
```

### Equality compares references

```js
console.log({} === {});           // false — two different objects
console.log([1, 2] === [1, 2]);   // false
```

### `const` does NOT make objects immutable

```js
const user = { name: "Asha" };
user.name = "Ravi"; // ✅ allowed — mutating the object
user = {};          // ❌ TypeError — reassigning the binding
```

### Passing to functions: "pass by sharing"

```js
function update(obj) {
  obj.x = 1;     // mutates the caller's object
  obj = { x: 2 }; // reassigns the local parameter only
}

const o = {};
update(o);
console.log(o); // { x: 1 }
```

> JS is always **pass-by-value**; for objects, the value being passed is a reference.

---

# 2. Property Access

```js
const user = { name: "Asha", "first-name": "A", 1: "one" };

user.name;          // dot notation — key must be a valid identifier
user["first-name"]; // bracket notation — any string
user[1];            // number keys are converted to strings → "1"

const key = "name";
user[key];          // dynamic key → "Asha"
user.key;           // undefined — looks for a property literally called "key"
```

### Property keys are only strings or symbols

Anything else is **converted to a string**.

```js
const cache = {};
const k1 = {};
const k2 = { a: 1 };

cache[k1] = "first";
cache[k2] = "second";

console.log(cache[k1]); // "second" ❗
console.log(Object.keys(cache)); // ["[object Object]"]
```

Both objects become the key `"[object Object]"`. **Use a `Map` for object keys** (see `17-map-set-weakmap-weakset.md`).

### Checking existence

```js
const obj = { a: undefined };

"a" in obj;                  // true — own OR inherited
Object.hasOwn(obj, "a");     // true — own only (modern)
obj.hasOwnProperty("a");     // true — own only (older)
obj.a !== undefined;         // false ❗ can't distinguish "missing" from "undefined"
"toString" in obj;           // true — inherited from Object.prototype
Object.hasOwn(obj, "toString"); // false
```

---

# 3. Modern Object Syntax

### Shorthand properties & methods

```js
const name = "Asha";
const age = 25;

const user = {
  name,           // same as name: name
  age,
  greet() {       // method shorthand
    return `Hi ${this.name}`;
  },
};
```

### Computed property names

```js
const field = "email";
const form = {
  [field]: "a@b.com",
  [`${field}Error`]: null,
};
// { email: "a@b.com", emailError: null }
```

Used constantly in React forms:

```js
setForm((prev) => ({ ...prev, [e.target.name]: e.target.value }));
```

### Destructuring

```js
const user = { name: "Asha", address: { city: "Pune" }, role: undefined };

const { name } = user;                       // basic
const { name: userName } = user;             // rename
const { role = "guest" } = user;             // default (applies only for undefined)
const { address: { city } } = user;          // nested
const { name: n, ...rest } = user;           // rest → { address, role }

// Function parameters
function greet({ name, greeting = "Hello" } = {}) {
  return `${greeting}, ${name}`;
}
```

> ⚠️ Defaults apply only when the value is `undefined`, **not** `null`.
>
> ```js
> const { x = 5 } = { x: null }; // x = null
> ```

> ⚠️ Destructuring `null`/`undefined` throws: `const { a } = null; // TypeError`. That's why `= {}` defaults on parameters are common.

### Spread & Rest

```js
const base = { a: 1, b: 2 };
const merged = { ...base, b: 3, c: 4 }; // { a: 1, b: 3, c: 4 } — later wins

const { a, ...others } = merged; // rest collects remaining own enumerable props
```

Spread copies **own enumerable** properties, **one level deep** (shallow).

---

# 4. `Object` Static Methods

```js
const user = { name: "Asha", age: 25 };

Object.keys(user);    // ["name", "age"]
Object.values(user);  // ["Asha", 25]
Object.entries(user); // [["name", "Asha"], ["age", 25]]
Object.fromEntries([["a", 1]]); // { a: 1 } — reverse of entries
```

### Transforming objects with entries

```js
const prices = { apple: 100, mango: 200 };

const discounted = Object.fromEntries(
  Object.entries(prices).map(([k, v]) => [k, v * 0.9]),
);
// { apple: 90, mango: 180 }
```

### `Object.assign`

```js
const target = { a: 1 };
const result = Object.assign(target, { b: 2 }, { c: 3 });

console.log(target);           // { a: 1, b: 2, c: 3 } — target is MUTATED
console.log(result === target); // true

const copy = Object.assign({}, target); // shallow copy pattern
```

| `Object.assign`              | Spread `{...}`            |
| ---------------------------- | ------------------------- |
| Mutates the target           | Always creates new object |
| Triggers **setters** on target | Defines new properties  |
| Shallow                      | Shallow                   |

### Key order ⭐ (output question favourite)

1. **Integer-like keys** in ascending numeric order
2. **String keys** in insertion order
3. Symbols are **excluded** from `keys/values/entries` and `for...in`

```js
Object.keys({ b: 1, 2: 1, a: 1, 1: 1 }); // ["1", "2", "b", "a"]
```

### Iteration differences

| Method                    | Own? | Inherited? | Non-enumerable? | Symbols? |
| ------------------------- | ---- | ---------- | --------------- | -------- |
| `for...in`                | ✅   | ✅ (enumerable) | ❌         | ❌       |
| `Object.keys`             | ✅   | ❌         | ❌              | ❌       |
| `Object.getOwnPropertyNames` | ✅ | ❌        | ✅              | ❌       |
| `Reflect.ownKeys`         | ✅   | ❌         | ✅              | ✅       |

---

# 5. Prototypes ⭐⭐⭐

## What is a Prototype?

Every object has a hidden internal link called **`[[Prototype]]`** that points to another object (or `null`). When you access a property that the object doesn't have, JavaScript **looks it up on the prototype**, then the prototype's prototype, and so on.

This chain is the **prototype chain**, and it's how JavaScript implements **inheritance**.

```js
const animal = {
  eats: true,
  walk() {
    return "walking";
  },
};

const dog = Object.create(animal); // dog.[[Prototype]] = animal
dog.barks = true;

dog.barks; // true  — own property
dog.eats;  // true  — found on animal
dog.walk(); // "walking"
dog.fly;   // undefined — reached null, not found
```

```text
dog { barks: true }
  │ [[Prototype]]
  ▼
animal { eats: true, walk() }
  │ [[Prototype]]
  ▼
Object.prototype { toString, hasOwnProperty, ... }
  │ [[Prototype]]
  ▼
null
```

### Lookup rules

- **Reading** walks up the chain until found or `null`.
- **Writing** (`obj.x = 1`) always creates/updates an **own** property — it does **not** modify the prototype (unless there's a setter up the chain).
- An own property with the same name **shadows** the prototype's.

```js
dog.eats = false;          // own property on dog — animal unchanged
console.log(animal.eats);  // true
delete dog.eats;
console.log(dog.eats);     // true — prototype's value visible again
```

---

## 5.1 `__proto__` vs `prototype` ⭐⭐⭐ (most confused topic)

| | `obj.__proto__` | `Fn.prototype` |
|---|---|---|
| Exists on | **every object** | **functions** (normal functions & classes) |
| What it is | Accessor for the object's `[[Prototype]]` — "who do I inherit from?" | The object that will become `[[Prototype]]` of instances created with `new Fn()` |
| Modern way | `Object.getPrototypeOf(obj)` / `Object.setPrototypeOf` | — |

```js
function Person(name) {
  this.name = name;
}
Person.prototype.greet = function () {
  return `Hi, I'm ${this.name}`;
};

const p = new Person("Asha");

Object.getPrototypeOf(p) === Person.prototype; // true
p.__proto__ === Person.prototype;              // true (legacy accessor)
Person.prototype.constructor === Person;       // true
p.greet();                                     // "Hi, I'm Asha"
```

```text
     Person (function)
        │ .prototype
        ▼
  Person.prototype { greet, constructor: Person }
        ▲
        │ [[Prototype]]
   p { name: "Asha" }
```

> Arrow functions **don't have** a `prototype` property and **can't** be used with `new`.

### Built-in chains

```js
const arr = [];
// arr → Array.prototype → Object.prototype → null

function fn() {}
// fn → Function.prototype → Object.prototype → null

Object.getPrototypeOf(Object.prototype); // null — end of every chain
```

### `Object.create`

```js
const proto = { hello() { return "hi"; } };
const obj = Object.create(proto);    // obj inherits from proto

const bare = Object.create(null);    // NO prototype at all
bare.toString;                       // undefined
// Useful as a pure dictionary — no inherited keys like "constructor"
```

### `Object.setPrototypeOf`

Changes an existing object's prototype. Works, but **slow** — engines optimize object shapes, and changing prototypes later de-optimizes them. Prefer `Object.create` at creation time.

---

# 6. Constructor Functions & `new` ⭐⭐⭐

Before classes, objects were created with **constructor functions** (capitalized by convention):

```js
function User(name) {
  this.name = name;
  // ❌ Don't define methods here — a new copy per instance
  // this.greet = function () {};
}

// ✅ Shared by all instances — defined once
User.prototype.greet = function () {
  return `Hi ${this.name}`;
};

const u1 = new User("Asha");
const u2 = new User("Ravi");
u1.greet === u2.greet; // true — same function, saves memory
```

### What `new` does internally ⭐⭐⭐

```text
new User("Asha")
  1. Create a new empty object
  2. Set its [[Prototype]] to User.prototype
  3. Call User with `this` bound to the new object
  4. If User returns an OBJECT → return that object
     Otherwise → return the new object
```

### Implement `new` ⭐⭐⭐

```js
function myNew(Constructor, ...args) {
  // Steps 1 + 2
  const obj = Object.create(Constructor.prototype);

  // Step 3
  const result = Constructor.apply(obj, args);

  // Step 4 — objects and functions override; primitives are ignored
  const isObject =
    result !== null && (typeof result === "object" || typeof result === "function");

  return isObject ? result : obj;
}

const u = myNew(User, "Asha");
u.greet();          // "Hi Asha"
u instanceof User;  // true
```

### Step 4 trap

```js
function A() {
  this.x = 1;
  return { y: 2 };
}
new A(); // { y: 2 } — returned object wins, `this` discarded

function B() {
  this.x = 1;
  return 42;
}
new B(); // B { x: 1 } — primitive return ignored
```

### Forgetting `new`

```js
function Car(model) { this.model = model; }

const c = Car("BMW"); // called as a normal function
// non-strict: `this` = globalThis → creates global `model` ❌
// strict:     `this` = undefined → TypeError
// c is undefined
```

Classes protect you: calling a class without `new` always throws.

---

# 7. Classes

Classes are mostly **syntactic sugar** over constructor functions + prototypes.

```js
class User {
  static count = 0;      // static field — on the class itself
  #password;             // private field — truly private
  role = "user";         // public instance field — own property on each instance

  constructor(name, password) {
    this.name = name;
    this.#password = password;
    User.count++;
  }

  greet() {              // goes on User.prototype (shared)
    return `Hi ${this.name}`;
  }

  get initials() {       // getter — accessed like a property
    return this.name[0];
  }

  set nickname(v) {      // setter
    this._nick = v.trim();
  }

  checkPassword(p) {
    return this.#password === p;
  }

  static create(name) {  // static method — User.create(), not instance.create()
    return new User(name, "temp");
  }
}

const u = new User("Asha", "secret");
u.initials;              // "A"
u.#password;             // ❌ SyntaxError outside the class
typeof User;             // "function"
```

### Class vs constructor function — differences that get asked

| Feature                  | Class                        | Constructor function |
| ------------------------ | ---------------------------- | -------------------- |
| Hoisting                 | Hoisted but in **TDZ** → can't use before declaration | Fully hoisted |
| Call without `new`       | ❌ TypeError                 | Runs (bug-prone)     |
| Strict mode              | Always strict inside         | Only if declared     |
| Methods enumerable?      | ❌ Non-enumerable            | ✅ (if assigned)     |
| Private fields           | `#field`                     | Only via closures    |

### Inheritance: `extends` & `super`

```js
class Animal {
  constructor(name) {
    this.name = name;
  }
  speak() {
    return `${this.name} makes a sound`;
  }
}

class Dog extends Animal {
  constructor(name, breed) {
    super(name);          // ⚠️ MUST call before using `this`
    this.breed = breed;
  }

  speak() {               // method overriding
    return `${super.speak()} — Woof!`; // call parent version
  }
}

const d = new Dog("Rex", "Lab");
d.speak(); // "Rex makes a sound — Woof!"
```

```text
d → Dog.prototype → Animal.prototype → Object.prototype → null
Dog.__proto__ === Animal   // static inheritance too
```

> Using `this` before `super()` in a derived constructor → `ReferenceError`.

### Arrow functions as class fields

```js
class Button {
  label = "Save";
  handleClick = () => console.log(this.label); // own property, `this` locked to instance
}
```

✅ Safe to pass as a callback (no lost `this`). ❌ A new function per instance (not on prototype) and can't be overridden via `super`.

---

# 8. Property Descriptors

Every property has hidden attributes:

| Attribute      | Meaning                                   |
| -------------- | ----------------------------------------- |
| `value`        | The value                                 |
| `writable`     | Can the value change?                     |
| `enumerable`   | Shows up in `for...in` / `Object.keys`?   |
| `configurable` | Can it be deleted / its descriptor changed? |
| `get` / `set`  | Accessor functions (instead of value/writable) |

```js
const obj = { a: 1 };
Object.getOwnPropertyDescriptor(obj, "a");
// { value: 1, writable: true, enumerable: true, configurable: true }
```

### `Object.defineProperty` — defaults are `false` ⚠️

```js
const user = {};
Object.defineProperty(user, "id", { value: 42 });

user.id = 100;          // silently ignored (TypeError in strict mode)
Object.keys(user);      // [] — not enumerable
delete user.id;         // false — not configurable
```

### Getter/setter via defineProperty

```js
const temp = { _c: 0 };
Object.defineProperty(temp, "f", {
  get() { return this._c * 9 / 5 + 32; },
  set(v) { this._c = (v - 32) * 5 / 9; },
  enumerable: true,
});
temp.f = 212;
temp._c; // 100
```

> Vue 2's reactivity was built on `Object.defineProperty` getters/setters (Vue 3 uses `Proxy`).

### `freeze` vs `seal` vs `preventExtensions`

| Action               | `preventExtensions` | `seal` | `freeze` |
| -------------------- | ------------------- | ------ | -------- |
| Add properties       | ❌                  | ❌     | ❌       |
| Delete properties    | ✅                  | ❌     | ❌       |
| Modify values        | ✅                  | ✅     | ❌       |
| Check                | `isExtensible`      | `isSealed` | `isFrozen` |

```js
const config = Object.freeze({ api: { url: "/v1" } });
config.api = null;      // ignored (TypeError in strict)
config.api.url = "/v2"; // ✅ works — freeze is SHALLOW ❗
```

Deep freeze:

```js
function deepFreeze(obj) {
  Object.values(obj).forEach((v) => {
    if (v && typeof v === "object" && !Object.isFrozen(v)) deepFreeze(v);
  });
  return Object.freeze(obj);
}
```

---

# 9. `instanceof` & Custom Implementation ⭐⭐⭐

`a instanceof B` checks: **is `B.prototype` anywhere in `a`'s prototype chain?**

```js
[] instanceof Array;   // true
[] instanceof Object;  // true — Array.prototype → Object.prototype
```

### Implement `instanceof`

```js
function myInstanceOf(obj, Constructor) {
  // Primitives are never instances
  if (obj === null || (typeof obj !== "object" && typeof obj !== "function")) {
    return false;
  }

  let proto = Object.getPrototypeOf(obj);
  const target = Constructor.prototype;

  while (proto !== null) {
    if (proto === target) return true;
    proto = Object.getPrototypeOf(proto);
  }
  return false;
}

myInstanceOf([], Array);    // true
myInstanceOf([], Object);   // true
myInstanceOf(5, Number);    // false
myInstanceOf(Object.create(null), Object); // false
```

### `instanceof` gotchas

- Works on the **chain**, not on who constructed the object — changing `prototype` later breaks it (see Q1 below).
- Fails across **iframes/realms** (different `Array` constructors) → use `Array.isArray`.
- Customisable via `static [Symbol.hasInstance]`.

---

# 10. Prototype-Based Inheritance (pre-class)

```js
function Animal(name) {
  this.name = name;
}
Animal.prototype.speak = function () {
  return `${this.name} makes a sound`;
};

function Dog(name, breed) {
  Animal.call(this, name);   // 1. inherit instance properties (like super())
  this.breed = breed;
}

Dog.prototype = Object.create(Animal.prototype); // 2. inherit methods
Dog.prototype.constructor = Dog;                  // 3. fix constructor reference

Dog.prototype.bark = function () {
  return "Woof";
};

const d = new Dog("Rex", "Lab");
d.speak();          // "Rex makes a sound"
d instanceof Animal; // true
```

> ❌ `Dog.prototype = Animal.prototype` — shares the same object; adding `bark` would add it to all animals.
> ❌ `Dog.prototype = new Animal()` — runs the parent constructor with no args and puts instance props on the prototype.

---

# 11. Output Questions 🧪

### Q1

```js
function Person() {}
Person.prototype.greet = () => "hi";

const p = new Person();
Person.prototype = { greet: () => "hello" };

console.log(p.greet());
console.log(p instanceof Person);
```

<details><summary>Answer</summary>

```text
hi
false
```

`p` still links to the **old** prototype object. `instanceof` checks the **current** `Person.prototype`, which isn't in `p`'s chain anymore.

</details>

### Q2

```js
const a = { x: 1 };
const b = Object.create(a);
b.x = 2;
delete b.x;
console.log(b.x);
delete b.x;
console.log(b.x);
```

<details><summary>Answer</summary>

```text
1
1
```

`delete` only removes **own** properties; it never touches the prototype.

</details>

### Q3

```js
const o = {};
const k1 = {};
const k2 = { a: 1 };
o[k1] = 1;
o[k2] = 2;
console.log(o[k1]);
```

<details><summary>Answer</summary>

`2` — both keys become `"[object Object]"`.

</details>

### Q4

```js
console.log(Object.keys({ b: 1, 2: 1, a: 1, 1: 1 }));
```

<details><summary>Answer</summary>

`["1", "2", "b", "a"]` — integer keys ascending first, then strings in insertion order.

</details>

### Q5

```js
const obj = new A();
class A {}
```

<details><summary>Answer</summary>

`ReferenceError: Cannot access 'A' before initialization` — classes are in the TDZ until declared.

</details>

### Q6

```js
function F() {
  this.a = 1;
  return { b: 2 };
}
const f = new F();
console.log(f.a, f.b, f instanceof F);
```

<details><summary>Answer</summary>

`undefined 2 false` — the returned object replaces `this`.

</details>

### Q7

```js
console.log(typeof class {});
console.log(Function instanceof Object, Object instanceof Function);
console.log(Object.getPrototypeOf(Object.prototype));
```

<details><summary>Answer</summary>

```text
function
true true
null
```

`Function` is an object; `Object` is a function. Every chain ends at `null`.

</details>

### Q8

```js
"use strict";
const cfg = Object.freeze({ level: 1, nested: { v: 1 } });
cfg.nested.v = 2;
console.log(cfg.nested.v);
cfg.level = 2;
```

<details><summary>Answer</summary>

Logs `2`, then throws `TypeError: Cannot assign to read only property 'level'`. Freeze is shallow; in strict mode writes to frozen props throw.

</details>

### Q9

```js
const proto = { items: [] };
const a = Object.create(proto);
const b = Object.create(proto);
a.items.push(1);
console.log(b.items);
```

<details><summary>Answer</summary>

`[1]` — `a.items` **reads** the array from the prototype, then mutates it. No assignment happened, so no own property was created. Shared mutable state on prototypes is a classic bug.

</details>

---

# 12. Interview Answers

**Prototype / prototype chain**

> "Every JavaScript object has an internal `[[Prototype]]` link to another object. When a property isn't found on the object, JavaScript looks it up along this chain until it finds it or reaches `null`. That's how inheritance works — methods live on a shared prototype rather than being copied to every instance."

**`__proto__` vs `prototype`**

> "`prototype` is a property on constructor functions and classes — it's the object that instances created with `new` will inherit from. `__proto__` is an accessor on every object that exposes its actual `[[Prototype]]`. So `new Person().__proto__ === Person.prototype`. Today we use `Object.getPrototypeOf` instead of `__proto__`."

**How `new` works**

> "`new` creates an empty object, links its prototype to the constructor's `prototype`, calls the constructor with `this` set to that object, and returns it — unless the constructor explicitly returns another object."

**Are classes real classes?**

> "Mostly syntactic sugar over prototypes — methods still go on `Class.prototype`. But they add real differences: they must be called with `new`, are always strict, are in the TDZ before declaration, have non-enumerable methods, and support private `#fields`."

---

# 13. Key Takeaways

- Objects are held by reference; `===` compares references.
- Property keys are strings or symbols — objects as keys collapse to `"[object Object]"`.
- Reading walks the prototype chain; writing creates own properties.
- `Fn.prototype` → becomes instances' `[[Prototype]]`; `__proto__` → an object's own link.
- `new` = create → link prototype → call with `this` → return object.
- Classes are sugar with stricter rules + private fields.
- `defineProperty` defaults everything to `false`.
- `freeze` is shallow; `seal` allows modifying values.
- `instanceof` walks the chain looking for `Constructor.prototype`.

## Readiness Checklist

- [ ] Draw the prototype chain for an array, a function, and a class instance
- [ ] Explain `__proto__` vs `prototype` in 30 seconds
- [ ] Implement `myNew` and `myInstanceOf` from memory
- [ ] Set up inheritance with constructor functions and with `class`
- [ ] Explain freeze vs seal vs preventExtensions
- [ ] Solve all 9 output questions
