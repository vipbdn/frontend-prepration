# 16 — JavaScript Modules

> **Interview + Understanding Guide** — `00-starter-roadmap.md` Section 16
>
> Goal: Understand ES Modules deeply (live bindings, static structure, hoisting), compare them with CommonJS, and explain how bundlers use modules for code splitting and tree shaking.

---

# 1. Why Modules?

Before modules, all scripts shared **one global scope**:

```html
<script src="utils.js"></script>  <!-- defines global `format` -->
<script src="app.js"></script>    <!-- hopes `format` exists; order matters; name clashes -->
```

Problems: global pollution, name collisions, implicit dependencies, load-order bugs. Workarounds: IIFEs and the "module pattern" (closures).

**Modules** give each file its **own scope**, explicit `import`/`export`, and a dependency graph tools can analyze.

---

# 2. ES Modules (ESM) ⭐

The official JavaScript module system (ES2015). Used natively by browsers and Node, and by all modern bundlers.

## 2.1 Named exports

```js
// math.js
export const PI = 3.14159;
export function add(a, b) {
  return a + b;
}
const secret = 42; // not exported → private to the module
export { secret as answer }; // export with a different name
```

```js
// app.js
import { PI, add } from "./math.js";
import { add as sum } from "./math.js"; // rename on import
import * as math from "./math.js";      // namespace object: math.PI, math.add
```

## 2.2 Default export

```js
// Button.js
export default function Button() { /* ... */ }
```

```js
import Button from "./Button.js";  // any name works
import MyButton from "./Button.js"; // same thing
import Button2, { size } from "./Button.js"; // default + named together
```

### Named vs Default — which to use?

| Named | Default |
| --- | --- |
| Many per module | One per module |
| Import name must match (or use `as`) | Importer picks any name |
| Better autocomplete, refactoring, tree shaking clarity | Common for "one thing per file" (React components) |
| Typos caught at build time | Inconsistent names across codebase |

Many teams prefer **named exports** for consistency. Both work.

## 2.3 Re-exports (barrel files)

```js
// components/index.js
export { Button } from "./Button.js";
export { Modal } from "./Modal.js";
export * from "./icons.js";
export { default as Card } from "./Card.js";
```

```js
import { Button, Modal, Card } from "./components";
```

⚠️ Large barrel files can hurt dev-server performance and sometimes tree shaking (if modules have side effects).

## 2.4 Side-effect imports

```js
import "./polyfills.js"; // runs the module, imports nothing
import "./styles.css";   // bundler feature
```

## 2.5 Using ESM in the browser

```html
<script type="module" src="app.js"></script>
```

Module scripts are:
- **Deferred** by default (execute after HTML parsing, in order)
- **Strict mode** automatically
- Top-level `this` is **`undefined`**
- Have their **own scope** (top-level variables aren't globals)
- Executed **once**, even if imported many times
- Loaded with **CORS** (need a server; `file://` won't work)
- Support **top-level `await`**

In Node: use `.mjs` or `"type": "module"` in `package.json`.

---

# 3. Key ESM Characteristics ⭐⭐⭐

## 3.1 Static structure

`import`/`export` must be at the **top level**, with **string-literal** paths. Can't be inside `if` or functions.

```js
if (cond) {
  import { a } from "./a.js"; // ❌ SyntaxError
}
```

This lets tools build the dependency graph **without running code** → enables **tree shaking**, early errors for missing exports, and faster loading.

## 3.2 Imports are hoisted

```js
console.log(add(1, 2)); // ✅ works — imports are processed before the module body runs
import { add } from "./math.js";
```

## 3.3 Live bindings (not copies) ⭐

Imports are **live, read-only views** of the exported variables.

```js
// counter.js
export let count = 0;
export function increment() {
  count++;
}
```

```js
// app.js
import { count, increment } from "./counter.js";

console.log(count); // 0
increment();
console.log(count); // 1 ✅ live — sees the updated value

count = 10; // ❌ TypeError: Assignment to constant variable — imports are read-only
```

Compare CommonJS, where you get a **copy** of the value at the time of `require` (for primitives).

## 3.4 Singletons

A module is evaluated **once**; every importer gets the **same instance**. Module-level state is shared:

```js
// store.js
export const store = { user: null }; // same object for all importers
```

---

# 4. Dynamic Imports ⭐

`import()` is a **function-like expression** that loads a module **at runtime** and returns a **promise**.

```js
button.addEventListener("click", async () => {
  const { openEditor } = await import("./editor.js"); // loaded only when needed
  openEditor();
});

// Conditional / computed paths
const locale = navigator.language.startsWith("hi") ? "hi" : "en";
const messages = await import(`./locales/${locale}.js`);

// Default export via dynamic import
const module = await import("./Chart.js");
const Chart = module.default;
```

Works in classic scripts too (not just modules). Bundlers turn each `import()` into a **separate chunk**.

---

# 5. Code Splitting ⭐

**Code splitting** = breaking the bundle into smaller chunks loaded **on demand**, so the initial download is smaller → faster first load.

```text
Without splitting:  main.js (2 MB) ──────────────▶ slow first paint
With splitting:     main.js (300 KB) → fast first paint
                    admin.chunk.js   → loaded when visiting /admin
                    chart.chunk.js   → loaded when opening a chart
```

### Strategies

- **Route-based** — each page is its own chunk (most impact)
- **Component-based** — heavy widgets (editors, charts, maps, modals)
- **Vendor splitting** — separate rarely-changing libraries for better caching

### React

```jsx
import { lazy, Suspense } from "react";

const AdminPage = lazy(() => import("./pages/AdminPage")); // must be a default export

function App() {
  return (
    <Suspense fallback={<Spinner />}>
      <Routes>
        <Route path="/admin" element={<AdminPage />} />
      </Routes>
    </Suspense>
  );
}
```

**Prefetching:** start loading likely-needed chunks early (`<link rel="prefetch">`, on hover, router prefetch) to hide latency.

---

# 6. Tree Shaking ⭐

**Tree shaking** = removing **unused exports** from the final bundle (dead-code elimination), based on ESM's **static** imports.

```js
// utils.js
export function used() { /* ... */ }
export function unused() { /* ... */ } // never imported anywhere
```

```js
import { used } from "./utils.js"; // bundler drops `unused` in production
```

### What prevents tree shaking

| Problem | Why |
| --- | --- |
| CommonJS (`require`, `module.exports`) | Dynamic — can't be statically analyzed |
| Side effects at module top level | Bundler must keep code that might have effects (`window.x = ...`, polyfills) |
| Importing whole libraries | `import _ from "lodash"` pulls everything → use `lodash-es` or `import debounce from "lodash/debounce"` |
| Namespace objects used dynamically | `utils[name]()` |
| Classes with unused methods | Methods on a used class are usually kept |

### `sideEffects` in `package.json`

```json
{
  "sideEffects": false
}
```

Tells bundlers "my modules have no side effects — safe to drop any unused module entirely". Or list files that do: `"sideEffects": ["*.css", "./src/polyfills.js"]`.

---

# 7. CommonJS (CJS)

Node's original module system.

```js
// math.js
const PI = 3.14;
function add(a, b) { return a + b; }
module.exports = { PI, add };
// or: exports.add = add;
```

```js
// app.js
const { add, PI } = require("./math");
const fs = require("fs");
```

### `exports` vs `module.exports` gotcha

```js
exports.a = 1;            // ✅ adds to module.exports
exports = { a: 1 };       // ❌ reassigns the local variable only — nothing exported
module.exports = { a: 1 }; // ✅ replaces the export
```

`exports` is just a reference to `module.exports`.

### CJS characteristics

- `require` is a **function** — can be called anywhere, conditionally, with computed paths
- **Synchronous** loading (fine for local disk on servers, bad for browsers)
- Exports are **copies of values** at require time (not live bindings)
- Module wrapper gives `module`, `exports`, `require`, `__filename`, `__dirname`
- Cached after first `require`

---

# 8. ES Modules vs CommonJS ⭐⭐⭐

| | ES Modules | CommonJS |
| --- | --- | --- |
| Syntax | `import` / `export` | `require` / `module.exports` |
| Structure | **Static** (top-level only) | **Dynamic** (anywhere) |
| Loading | Async-capable; designed for browsers | Synchronous |
| Bindings | **Live**, read-only | Value **copies** |
| Tree shaking | ✅ | ❌ (mostly) |
| Top-level `await` | ✅ | ❌ |
| `this` at top level | `undefined` | `module.exports` |
| Strict mode | Always | Only if declared |
| Environment | Browsers, Node, bundlers | Node (bundlers can convert) |
| File ext (Node) | `.mjs` or `"type": "module"` | `.cjs` or default |
| Dynamic loading | `import()` | `require()` |

### Interop

- ESM can `import` CJS (the default import is `module.exports`).
- CJS can't `require` ESM synchronously in older Node (use `await import()`); newer Node versions support `require(esm)` for modules without top-level await.

### Live binding vs copy example

```js
// CJS counter.cjs
let count = 0;
module.exports = { count, increment: () => count++ };

// app.cjs
const c = require("./counter.cjs");
c.increment();
console.log(c.count); // 0 ❗ copy taken at export time

// ESM version → 1 (live)
```

---

# 9. Circular Dependencies

```text
a.js ──imports──▶ b.js
  ▲                 │
  └────imports──────┘
```

```js
// a.js
import { b } from "./b.js";
export const a = "A";
console.log("a sees", b);

// b.js
import { a } from "./a.js";
export const b = "B";
console.log("b sees", a); // ❌ ReferenceError: Cannot access 'a' before initialization
```

Entry `a.js` → starts loading `b.js` first → `b` runs while `a` hasn't initialized its exports yet (TDZ).

- In **ESM**: accessing an uninitialized `let/const` binding → `ReferenceError`; **function declarations** are hoisted and work.
- In **CJS**: you get a **partially filled** `exports` object (often `undefined` values).

### Fixes

- Extract shared code into a **third module** both import
- Access imports **lazily** (inside functions called later), not at top level
- Restructure dependencies (often a design smell)
- Tools: `madge`, ESLint `import/no-cycle`

---

# 10. Output Questions 🧪

### Q1

```js
// counter.js
export let n = 1;
export const inc = () => n++;

// main.js
import { n, inc } from "./counter.js";
inc();
inc();
console.log(n);
```

<details><summary>Answer</summary>

`3` — live binding.

</details>

### Q2

```js
// main.js (ESM)
console.log(this);
console.log(typeof window === "undefined" ? "node" : "browser");
```

<details><summary>Answer</summary>

`undefined` — module top-level `this` is `undefined`.

</details>

### Q3

```js
// log.js
console.log("log module evaluated");
export const log = (m) => console.log(m);

// a.js
import { log } from "./log.js";
// b.js
import { log } from "./log.js";
// main.js
import "./a.js";
import "./b.js";
```

<details><summary>Answer</summary>

`log module evaluated` — printed **once**. Modules are evaluated once and cached.

</details>

---

# 11. Interview Answers

**ESM vs CommonJS**

> "ES Modules use static `import`/`export` at the top level, so tools can analyze dependencies without running code — that enables tree shaking. Imports are live read-only bindings, they're always strict, support top-level `await`, and work natively in browsers. CommonJS uses `require` and `module.exports`, loads synchronously, can be called dynamically anywhere, and exports copies of values. Node supports both; frontend tooling is built around ESM."

**Code splitting vs tree shaking**

> "Tree shaking removes code that's never used from the bundle. Code splitting doesn't remove code — it splits it into chunks loaded on demand, usually with dynamic `import()` and `React.lazy`. Tree shaking makes the total smaller; code splitting makes the initial load smaller."

---

# 12. Key Takeaways

- Modules = file scope + explicit imports/exports.
- ESM: static, hoisted imports, live read-only bindings, singletons, strict, deferred.
- Named vs default exports — know the trade-offs.
- `import()` = dynamic, returns a promise → code splitting.
- Tree shaking needs static ESM + no side effects (`sideEffects` flag).
- CJS: dynamic `require`, sync, value copies; `exports` vs `module.exports` gotcha.
- Circular deps → TDZ errors in ESM, partial exports in CJS; extract shared code.

## Readiness Checklist

- [ ] Explain live bindings with an example
- [ ] Compare ESM vs CJS in a table from memory
- [ ] Implement route-based code splitting with `React.lazy`
- [ ] Explain what breaks tree shaking
- [ ] Explain and fix a circular dependency
