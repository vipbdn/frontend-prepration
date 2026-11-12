# 19 — Symbols

> **Interview + Understanding Guide** — `00-starter-roadmap.md` Section 19
>
> Goal: Understand what symbols are, why they exist, how they behave as object keys, and the well-known symbols (`Symbol.iterator`, `Symbol.toPrimitive`, `Symbol.toStringTag`) that let you customise language behaviour.

---

# 1. What is a Symbol?

A **Symbol** is a **primitive** type (ES2015) whose every value is **unique** and immutable. Its main purpose: creating **property keys that can't collide** with any other key.

```js
const id1 = Symbol("id"); // "id" is just a description (for debugging)
const id2 = Symbol("id");

id1 === id2;      // false — every Symbol() is unique
typeof id1;       // "symbol"
id1.toString();   // "Symbol(id)"
id1.description;  // "id"
```

- Created with `Symbol()` — **not** `new Symbol()` (TypeError: not a constructor).
- No literal syntax.
- Can't be implicitly converted to a string:

```js
`${id1}`;      // ❌ TypeError: Cannot convert a Symbol value to a string
"" + id1;      // ❌ TypeError
String(id1);   // ✅ "Symbol(id)"
```

---

# 2. Unique Symbols as Object Keys ⭐

```js
const ID = Symbol("id");

const user = {
  name: "Asha",
  [ID]: 123, // computed key
};

user[ID];   // 123
user.ID;    // undefined — that's the string key "ID"
```

### Why use symbol keys?

1. **No collisions** — a library can attach data to your objects without overwriting your keys (and you can't accidentally overwrite theirs).
2. **"Hidden" from normal iteration** — not shown by `for...in`, `Object.keys`, `JSON.stringify`.

```js
const meta = Symbol("meta");
const obj = { a: 1, [meta]: "internal" };

Object.keys(obj);                  // ["a"]
JSON.stringify(obj);               // '{"a":1}'
for (const k in obj) console.log(k); // "a"
Object.assign({}, obj);            // ✅ copies symbol props too
({ ...obj });                      // ✅ spread copies symbol props

// But they're NOT private:
Object.getOwnPropertySymbols(obj); // [Symbol(meta)]
Reflect.ownKeys(obj);              // ["a", Symbol(meta)]
```

> Symbols are **not a privacy mechanism** — use `#private` fields or closures/WeakMaps for that. They're a **collision-avoidance** mechanism.

### Symbols as enum-like constants

```js
const Status = Object.freeze({
  IDLE: Symbol("idle"),
  LOADING: Symbol("loading"),
  ERROR: Symbol("error"),
});

if (state === Status.LOADING) { /* ... */ }
// Can't be faked with a string "loading"
```

(React uses symbols internally, e.g. `$$typeof: Symbol.for("react.element")`, so JSON from a server can't be injected as a fake React element — a security measure against XSS.)

---

# 3. Global Symbol Registry

`Symbol.for(key)` returns a symbol from a **global, cross-realm registry** — the same key always gives the **same** symbol.

```js
const a = Symbol.for("app.id");
const b = Symbol.for("app.id");
a === b;                 // true ✅

Symbol("app.id") === Symbol.for("app.id"); // false

Symbol.keyFor(a);        // "app.id"
Symbol.keyFor(Symbol("x")); // undefined — not in registry
```

Use when different parts of an app (or different iframes/realms) must agree on the same symbol. Namespace keys to avoid clashes (`"mylib.something"`).

| `Symbol(desc)` | `Symbol.for(key)` |
| --- | --- |
| Always new and unique | Same symbol for the same key, globally |
| Description is just a label | Key is the lookup identity |

---

# 4. Well-Known Symbols ⭐

Built-in symbols that let objects **hook into language behaviour**.

## 4.1 `Symbol.iterator`

Makes an object iterable (`for...of`, spread, destructuring). See `18-iterators-and-generators.md`.

```js
const team = {
  members: ["Asha", "Ravi"],
  *[Symbol.iterator]() {
    yield* this.members;
  },
};
[...team]; // ["Asha", "Ravi"]
```

`Symbol.asyncIterator` does the same for `for await...of`.

## 4.2 `Symbol.toPrimitive` ⭐

Controls how an object converts to a primitive. Receives a **hint**: `"number"`, `"string"` or `"default"`.

```js
const money = {
  amount: 100,
  currency: "INR",
  [Symbol.toPrimitive](hint) {
    if (hint === "number") return this.amount;
    if (hint === "string") return `${this.amount} ${this.currency}`;
    return this.amount; // "default" — used by + and ==
  },
};

+money;          // 100           (hint: number)
`${money}`;      // "100 INR"     (hint: string)
money + 50;      // 150           (hint: default)
money * 2;       // 200           (hint: number)
money == 100;    // true          (hint: default)
String(money);   // "100 INR"
```

Without `Symbol.toPrimitive`, JS uses `valueOf()` then `toString()` (or the reverse for string hint). This ties directly to **type coercion** in `01-javascript-fundamentals-and-internals.md`.

## 4.3 `Symbol.toStringTag`

Customises `Object.prototype.toString` output.

```js
class Queue {
  get [Symbol.toStringTag]() {
    return "Queue";
  }
}

Object.prototype.toString.call(new Queue()); // "[object Queue]"
Object.prototype.toString.call([]);          // "[object Array]"
Object.prototype.toString.call(null);        // "[object Null]"
```

That `Object.prototype.toString.call(x)` trick is a reliable **type check**:

```js
const typeOf = (x) => Object.prototype.toString.call(x).slice(8, -1);
typeOf([]);          // "Array"
typeOf(new Date());  // "Date"
typeOf(null);        // "Null"
typeOf(/a/);         // "RegExp"
```

## 4.4 Others (know they exist)

| Symbol | Customises |
| --- | --- |
| `Symbol.hasInstance` | `instanceof` behaviour |
| `Symbol.asyncIterator` | `for await...of` |
| `Symbol.isConcatSpreadable` | Whether `concat` spreads the object |
| `Symbol.species` | Constructor used for derived objects (`map` on subclasses) |
| `Symbol.match/replace/search/split` | How objects behave in string methods |
| `Symbol.unscopables` | `with` statement exclusions |

```js
class Even {
  static [Symbol.hasInstance](n) {
    return Number.isInteger(n) && n % 2 === 0;
  }
}
4 instanceof Even; // true
```

---

# 5. Output Questions 🧪

```js
const s1 = Symbol("x");
const s2 = Symbol("x");
console.log(s1 === s2, Symbol.for("x") === Symbol.for("x")); // Q1

const k = Symbol("k");
const o = { [k]: 1, a: 2 };
console.log(Object.keys(o), JSON.stringify(o));             // Q2
console.log(Reflect.ownKeys(o).length);                      // Q3

try { console.log(`${k}`); } catch (e) { console.log(e.name); } // Q4

const obj = {
  [Symbol.toPrimitive](hint) { return hint === "number" ? 42 : "hello"; },
};
console.log(+obj, `${obj}`, obj + "");                       // Q5

console.log(typeof Symbol(), typeof Symbol);                 // Q6
```

<details><summary>Answers</summary>

1. `false true`
2. `["a"] {"a":2}`
3. `2`
4. `TypeError`
5. `42 hello hello` — `obj + ""` uses hint `"default"` → `"hello"`.
6. `symbol function`

</details>

---

# 6. Interview Answers

**What are symbols used for?**

> "A Symbol is a unique, immutable primitive mainly used as an object property key that can't collide with other keys — useful for libraries attaching metadata to objects. Symbol keys are skipped by `for...in`, `Object.keys` and JSON, but they're not private. JavaScript also has well-known symbols like `Symbol.iterator` to make objects iterable and `Symbol.toPrimitive` to control type conversion."

**`Symbol()` vs `Symbol.for()`**

> "`Symbol()` always creates a new unique symbol. `Symbol.for(key)` looks up a global registry and returns the same symbol for the same key everywhere, even across iframes."

---

# 7. Key Takeaways

- Symbols are unique primitives; `Symbol("desc")` — no `new`.
- Main use: collision-free property keys.
- Hidden from `for...in`, `Object.keys`, JSON — but visible via `getOwnPropertySymbols` / `Reflect.ownKeys`; **not private**.
- No implicit string conversion; use `String(sym)` or `.description`.
- `Symbol.for` = global registry (shared).
- Well-known symbols: `iterator`, `asyncIterator`, `toPrimitive`, `toStringTag`, `hasInstance`.

## Readiness Checklist

- [ ] Explain why symbols exist
- [ ] Show how symbol keys behave with keys/JSON/spread/Reflect.ownKeys
- [ ] Implement `Symbol.toPrimitive` and predict coercion results
- [ ] Use `Object.prototype.toString.call` for type checks
