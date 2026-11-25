# 🔷 The Complete TypeScript Guide — Daily Use, Revision & Interview Prep

> A practical guide you can **follow while coding**, **revise quickly**, and **prepare for interviews** with. Beginners can read it top to bottom; experienced developers can jump to generics, type-level programming, the cheat sheet, common errors or interview questions.
>
> Written for **TypeScript 5.x → 7.0** (TS 6.0 was the last JavaScript-based compiler; **TS 7.0 is the native Go port**, roughly 10× faster). Code examples in this guide were type-checked with the TypeScript compiler.

---

## 📖 How to Use This Guide

| If you want to… | Go to |
| --- | --- |
| Learn TypeScript from scratch | Sections 1 → 8 in order |
| Set up a project / understand `tsconfig` | [§2 Setup & tsconfig](#2-setup--tsconfig-) |
| Master generics & advanced types | [§9 Generics](#9-generics-), [§10 Type Operators](#10-type-operators--type-level-programming-), [§11 Utility Types](#11-utility-types-) |
| Fix a compiler error right now | [§19 Common Errors & Fixes](#19-common-errors--fixes-) |
| Look something up quickly | [§22 Cheat Sheet](#22-cheat-sheet-) |
| Prepare for an interview | [§23 Interview Questions](#23-interview-questions--answers-), [§24 Type Challenges](#24-type-challenges--exercises) |

**Legend:** ⭐ used daily · ❌ compile error (shown on purpose) · ⚠️ pitfall · 💡 tip

---

## 📑 Table of Contents

1. [What is TypeScript?](#1-what-is-typescript)
2. [Setup & tsconfig](#2-setup--tsconfig-)
3. [Basic Types](#3-basic-types-)
4. [Type Inference & Annotations](#4-type-inference--annotations)
5. [Functions](#5-functions-)
6. [Objects: Type Aliases vs Interfaces](#6-objects-type-aliases-vs-interfaces-)
7. [Unions, Intersections & Literal Types](#7-unions-intersections--literal-types-)
8. [Narrowing & Type Guards](#8-narrowing--type-guards-)
9. [Generics](#9-generics-)
10. [Type Operators & Type-Level Programming](#10-type-operators--type-level-programming-)
11. [Utility Types](#11-utility-types-)
12. [Classes](#12-classes)
13. [Enums & Alternatives](#13-enums--alternatives)
14. [Modules, Declarations & `.d.ts`](#14-modules-declarations--dts)
15. [Type Compatibility: any, unknown, never & Variance](#15-type-compatibility-any-unknown-never--variance-)
16. [Async Code & Error Handling](#16-async-code--error-handling)
17. [Runtime Validation & Branded Types](#17-runtime-validation--branded-types-)
18. [TypeScript with React & Node](#18-typescript-with-react--node)
19. [Common Errors & Fixes](#19-common-errors--fixes-)
20. [Best Practices](#20-best-practices-)
21. [Real-World Recipes](#21-real-world-recipes-)
22. [Cheat Sheet](#22-cheat-sheet-)
23. [Interview Questions & Answers](#23-interview-questions--answers-)
24. [Type Challenges & Exercises](#24-type-challenges--exercises)

---

# 1. What is TypeScript?

**TypeScript** is **JavaScript with a static type system**. You write `.ts` files with type annotations; the TypeScript compiler (`tsc`) **checks** your types and **outputs plain JavaScript**. Created by Microsoft (Anders Hejlsberg) in 2012.

```text
  app.ts  ──tsc (type-check + strip types)──▶  app.js  ──▶ runs in browser / Node
```

### Key facts ⭐

- TypeScript is a **superset** of JavaScript — any valid JS is (almost always) valid TS.
- Types exist **only at compile time**. They are **erased** — there are no types at runtime (no runtime type checks unless you add them, see §17).
- TypeScript checks your code; it **doesn't change behaviour** (except for a few features that emit code: `enum`, `namespace`, parameter properties, decorators).
- The type system is **structural** ("duck typing"), not nominal: if two types have the same shape, they're compatible.

### Why use it?

| Benefit | Example |
| --- | --- |
| Catch bugs before running | Typos, `undefined` access, wrong argument types |
| Superb editor tooling | Autocomplete, go-to-definition, safe rename/refactor |
| Self-documenting code | Function signatures describe inputs/outputs |
| Confident refactoring | Change a type → compiler shows every place to update |
| Scales to large teams/codebases | Contracts between modules, APIs and components |

### Trade-offs

- Build/setup step (mostly invisible today: Vite, Next.js, Node type-stripping)
- Learning curve for advanced types
- Types can give **false confidence** at boundaries (API responses, `JSON.parse`, `as` casts) → validate at runtime

### JavaScript vs TypeScript

```js
// JavaScript — bug found at runtime (maybe in production)
function getTotal(items) {
  return items.reduce((sum, i) => sum + i.price, 0);
}
getTotal([{ name: "Book", prcie: 10 }]); // NaN — typo not caught
```

```ts
// TypeScript — bug found while typing
type Item = { name: string; price: number };
function getTotal(items: Item[]): number {
  return items.reduce((sum, i) => sum + i.price, 0);
}
getTotal([{ name: "Book", prcie: 10 }]); // ❌ Object literal may only specify known properties
```

---

# 2. Setup & tsconfig ⭐

## 2.1 Install & run

```bash
npm install -D typescript          # per project (recommended)
npx tsc --init                     # create tsconfig.json
npx tsc                            # type-check + emit JS (per tsconfig)
npx tsc --noEmit                   # ⭐ type-check only (common in CI with bundlers)
npx tsc --watch                    # recompile on changes
npx tsc -p tsconfig.build.json     # use a specific config
```

### Running TypeScript directly

| Tool | Use |
| --- | --- |
| **Node.js type stripping** | Node 22.18+/23.6+ runs `.ts` files directly by stripping types (`node app.ts`). Only "erasable" syntax (no `enum`, `namespace`, parameter properties) — pair with `erasableSyntaxOnly` |
| **tsx** | `npx tsx watch src/index.ts` — fast dev runner (esbuild) |
| **ts-node** | Older runner with type-checking |
| **Bun / Deno** | Run TS natively |
| **Vite / Next.js / esbuild / SWC** | Transpile TS fast **without** type-checking → run `tsc --noEmit` separately |

> 💡 Bundlers and runners strip types but **don't type-check**. Always run `tsc --noEmit` in CI (and keep your editor showing errors).

### TypeScript 6.0 / 7.0 changes to know

- **`strict` is on by default** (no config needed for strict checks).
- **Deprecated in 6.0, removed in 7.0:** `target: "es5"`, `moduleResolution: "node"` (node10), `baseUrl`, `outFile`, AMD/UMD/System modules, `esModuleInterop: false`, `downlevelIteration`. In 6.0 you can temporarily silence them with `"ignoreDeprecations": "6.0"`.
- **TS 7.0** is a native (Go) rewrite: much faster builds and editor responsiveness; same language.

## 2.2 A modern `tsconfig.json` ⭐

```jsonc
{
  "compilerOptions": {
    /* Language & environment */
    "target": "ES2022",                 // JS version to emit
    "lib": ["ES2023", "DOM", "DOM.Iterable"], // built-in type definitions available
    "jsx": "react-jsx",                 // for React projects

    /* Modules */
    "module": "ESNext",                 // or "NodeNext" for Node packages, "preserve" for bundlers
    "moduleResolution": "bundler",      // "bundler" (Vite/webpack) or "nodenext" (Node)
    "resolveJsonModule": true,
    "verbatimModuleSyntax": true,       // imports/exports emitted exactly as written; forces `import type`
    "isolatedModules": true,            // each file transpilable on its own (esbuild/SWC/Babel)

    /* Type checking */
    "strict": true,                     // ⭐ enables the whole strict family (default in TS 6+)
    "noUncheckedIndexedAccess": true,   // ⭐ arr[i] / obj[key] include `undefined`
    "exactOptionalPropertyTypes": true, // `a?: string` ≠ `a: string | undefined`
    "noImplicitOverride": true,
    "noFallthroughCasesInSwitch": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true,

    /* Emit */
    "noEmit": true,                     // bundler emits JS; tsc only checks
    // "outDir": "dist", "declaration": true, "sourceMap": true   ← for libraries / Node builds

    /* Interop & performance */
    "skipLibCheck": true,               // skip type-checking .d.ts in node_modules (faster)
    "forceConsistentCasingInFileNames": true,

    /* Path aliases */
    "paths": { "@/*": ["./src/*"] }     // (bundler must be configured too); no baseUrl needed
  },
  "include": ["src"],
  "exclude": ["node_modules", "dist"]
}
```

### The `strict` family ⭐

| Flag | What it does |
| --- | --- |
| `strictNullChecks` | `null`/`undefined` are separate types — must be handled explicitly (most important) |
| `noImplicitAny` | Error when a type would silently become `any` |
| `strictFunctionTypes` | Function parameters checked contravariantly |
| `strictBindCallApply` | Type-check `call`, `bind`, `apply` |
| `strictPropertyInitialization` | Class properties must be initialized |
| `noImplicitThis` | Error on `this` with implicit `any` type |
| `useUnknownInCatchVariables` | `catch (e)` → `e: unknown` |
| `alwaysStrict` | Emit `"use strict"` |

### Common setups

| Project | Key settings |
| --- | --- |
| React + Vite | `module: ESNext`, `moduleResolution: bundler`, `jsx: react-jsx`, `noEmit: true` |
| Next.js | Generated by Next (`moduleResolution: bundler`, `jsx: preserve`, `plugins: [{ name: "next" }]`) |
| Node app (ESM) | `module: NodeNext`, `moduleResolution: NodeNext`, `outDir: dist`, `"type": "module"` in package.json |
| Library | `declaration: true`, `declarationMap: true`, `outDir`, consider `tsup`/`tsdown` for dual builds |
| Monorepo | `composite: true` + project `references`, `tsc -b` |

`extends` to share config: `"extends": "@tsconfig/strictest/tsconfig.json"` or a base config in your monorepo.

---

# 3. Basic Types ⭐

## 3.1 Primitives

```ts
let username: string = "Asha";
let age: number = 29;             // integers and floats
let isAdmin: boolean = false;
let big: bigint = 100n;
let id: symbol = Symbol("id");
let nothing: null = null;
let notSet: undefined = undefined;
```

⚠️ Use lowercase `string`, `number`, `boolean` — not the wrapper types `String`, `Number`, `Boolean`.

## 3.2 Arrays & Tuples

```ts
const scores: number[] = [90, 85];
const names: Array<string> = ["a", "b"];       // same thing, generic syntax
const matrix: number[][] = [[1, 2], [3, 4]];
const ids: readonly number[] = [1, 2, 3];       // ReadonlyArray<number>
ids.push(4);                                     // ❌ Property 'push' does not exist on type 'readonly number[]'

// Tuples: fixed length, typed positions
const point: [number, number] = [10, 20];
const entry: [string, number] = ["age", 29];
const user: [id: number, name: string, email?: string] = [1, "Asha"]; // labeled + optional element
const args: [string, ...number[]] = ["sum", 1, 2, 3];                // rest element

function useCounter(): [number, () => void] {
  let n = 0;
  return [n, () => n++];
}
const [count, increment] = useCounter();
```

## 3.3 Objects

```ts
const product: { id: number; name: string; price?: number } = { id: 1, name: "Pen" };
// price is optional (may be missing)
```

Usually you name object types with `type` or `interface` (§6).

## 3.4 Special types ⭐

| Type | Meaning | Use |
| --- | --- | --- |
| `any` | Turns off type checking ⚠️ | Migration only; avoid |
| `unknown` | "Some value, type not known yet" — must narrow before use | ✅ Safe alternative to `any` (API data, `catch`) |
| `never` | A value that **never** occurs | Exhaustive checks, functions that always throw, impossible states |
| `void` | No meaningful return value | Function return type |
| `object` | Any non-primitive | Rare |
| `{}` | Any non-null/undefined value ⚠️ (not "empty object") | Rare |

```ts
let a: any = "hello";
a.foo.bar();               // no error — crashes at runtime ⚠️

let u: unknown = "hello";
u.toUpperCase();           // ❌ 'u' is of type 'unknown'
if (typeof u === "string") {
  u.toUpperCase();         // ✅ narrowed to string
}

function fail(message: string): never {
  throw new Error(message); // never returns normally
}

function log(message: string): void {
  console.log(message);
}
```

## 3.5 Literal types & `as const`

```ts
let direction: "up" | "down" = "up";
direction = "left";               // ❌ Type '"left"' is not assignable to type '"up" | "down"'

const status = "active";          // type: "active" (const → literal)
let mutableStatus = "active";     // type: string (let → widened)

const config = { mode: "dark", retries: 3 } as const;
// type: { readonly mode: "dark"; readonly retries: 3 }

const ROLES = ["admin", "editor", "viewer"] as const;
type Role = (typeof ROLES)[number];   // "admin" | "editor" | "viewer"
```

---

# 4. Type Inference & Annotations

TypeScript **infers** types from values. Annotate where inference can't help or where you want a **contract**.

```ts
let count = 0;                 // inferred: number
const items = ["a", "b"];      // inferred: string[]
const user = { name: "Asha", age: 29 }; // inferred: { name: string; age: number }

function double(n: number) {   // return type inferred: number
  return n * 2;
}
```

### When to annotate ✅

- **Function parameters** (always — they can't be inferred from usage)
- **Exported/public function return types** (stable API, better errors, faster compiles)
- **Empty collections**: `const ids: number[] = []`
- **Variables initialized later**: `let user: User | null = null`
- When inference is **too wide or too narrow**

### When not to

```ts
const total: number = 5 + 10;   // redundant — let inference work
```

### Contextual typing

```ts
const names = ["a", "b"];
names.forEach((n) => n.toUpperCase());              // n inferred as string from the array

window.addEventListener("click", (e) => e.clientX); // e inferred as MouseEvent
```

### `satisfies` ⭐ (validate without widening)

```ts
type Theme = Record<"primary" | "secondary", string>;

const theme = {
  primary: "#4f46e5",
  secondary: "#64748b",
} satisfies Theme;            // checked against Theme…

theme.primary.toUpperCase();  // …but keeps the precise inferred type

const bad = { primary: "#fff" } satisfies Theme;
// ❌ Property 'secondary' is missing
```

| `const x: T = value` | `const x = value satisfies T` | `value as T` |
| --- | --- | --- |
| Type becomes `T` (widened) | Checked against `T`, keeps the inferred type | No check — you override the compiler ⚠️ |

---

# 5. Functions ⭐

## 5.1 Parameters & returns

```ts
function add(a: number, b: number): number {
  return a + b;
}

const multiply = (a: number, b: number): number => a * b;

function greet(name: string, greeting = "Hello", punctuation?: string): string {
  return `${greeting}, ${name}${punctuation ?? "!"}`;
}
greet("Asha");
greet("Asha", "Hi", ".");

function sum(...nums: number[]): number {
  return nums.reduce((s, n) => s + n, 0);
}

// Destructured parameters
function createUser({ name, age = 18 }: { name: string; age?: number }) {
  return { name, age };
}
```

## 5.2 Function types

```ts
type BinaryOp = (a: number, b: number) => number;
const subtract: BinaryOp = (a, b) => a - b;     // params inferred from the type

type Handler = (event: { type: string }) => void;

// Call signature in an object type (functions with properties)
type Formatter = {
  (value: number): string;
  locale: string;
};
```

### `void` return types are lenient

```ts
type Callback = () => void;
const cb: Callback = () => 42;   // ✅ allowed — return value is ignored
[1, 2].forEach((n) => [].push(n as never)); // forEach callbacks may return anything
```

But a function **declared** with `: void` can't return a value.

## 5.3 Overloads

Multiple call signatures for one implementation.

```ts
function parse(value: string): number;
function parse(value: number): string;
function parse(value: string | number): string | number {
  return typeof value === "string" ? Number(value) : String(value);
}

const n = parse("42"); // number
const s = parse(42);   // string
```

💡 Prefer **union parameters** or **generics** when possible; overloads are for when the return type depends on the input type in ways those can't express simply.

## 5.4 Typing `this`

```ts
interface Button {
  label: string;
  onClick(this: Button): void;
}

function handler(this: HTMLElement, e: MouseEvent) {
  this.classList.add("clicked");
}
```

## 5.5 Assertion functions & type predicates (preview)

```ts
function isString(value: unknown): value is string {
  return typeof value === "string";
}

function assertDefined<T>(value: T, msg = "Value is undefined"): asserts value is NonNullable<T> {
  if (value === undefined || value === null) throw new Error(msg);
}
```

Details in §8.

---

# 6. Objects: Type Aliases vs Interfaces ⭐

## 6.1 Both describe object shapes

```ts
type UserType = {
  id: number;
  name: string;
  email?: string;            // optional
  readonly createdAt: Date;  // can't be reassigned
};

interface UserInterface {
  id: number;
  name: string;
  email?: string;
  readonly createdAt: Date;
}
```

## 6.2 Extending

```ts
interface Animal {
  name: string;
}
interface Dog extends Animal {
  breed: string;
}

type Cat = Animal & { indoor: boolean };   // intersection

interface Admin extends Animal {   // interfaces can extend other interfaces (or object types)
  permissions: string[];
}
```

## 6.3 `type` vs `interface` ⭐⭐⭐ (interview favourite)

| | `interface` | `type` |
| --- | --- | --- |
| Object shapes | ✅ | ✅ |
| Unions (`A \| B`), primitives, tuples, function types | ❌ | ✅ |
| Mapped & conditional types | ❌ | ✅ |
| Extend | `extends` | `&` intersection |
| **Declaration merging** (same name declared twice merges) | ✅ | ❌ (duplicate identifier error) |
| Error messages / performance with large hierarchies | Slightly better (cached by name) | Fine for most code |
| Classes can `implements` | ✅ | ✅ (if object type) |

```ts
// Declaration merging — useful to augment library/global types
// (inside a module file, wrap global interfaces in `declare global { … }` — see §14.4)
declare global {
  interface Window {
    analytics: { track(event: string): void };
  }
}
window.analytics.track("signup");
export {};
```

**Rule of thumb:** use `type` by default (it can do everything); use `interface` for public object contracts that consumers may extend, or when augmenting libraries. Most importantly: **be consistent**.

## 6.4 Index signatures & `Record`

```ts
type Scores = { [studentName: string]: number };
const scores: Scores = { asha: 90, ravi: 85 };

type Dictionary = Record<string, number>;   // same idea

const s = scores["unknown"];   // number | undefined with noUncheckedIndexedAccess ✅
```

## 6.5 Excess property checks

Object **literals** assigned directly are checked for unknown properties:

```ts
type Point = { x: number; y: number };
const p: Point = { x: 1, y: 2, z: 3 };   // ❌ Object literal may only specify known properties

const temp = { x: 1, y: 2, z: 3 };
const q: Point = temp;                    // ✅ no error — not a fresh literal (structural typing)
```

## 6.6 Optional vs `undefined`

```ts
type A = { name?: string };              // property may be missing
type B = { name: string | undefined };   // property must exist (value may be undefined)

const a: A = {};       // ✅
const b: B = {};       // ❌ Property 'name' is missing
```

With `exactOptionalPropertyTypes`, `{ name: undefined }` isn't assignable to `A` either.

---

# 7. Unions, Intersections & Literal Types ⭐

## 7.1 Union `|` — "one of"

```ts
type Id = number | string;

function printId(id: Id) {
  console.log(id.toUpperCase());   // ❌ Property 'toUpperCase' does not exist on type 'number'
  if (typeof id === "string") console.log(id.toUpperCase()); // ✅ narrow first
}

type Status = "idle" | "loading" | "success" | "error";   // string literal union
```

On a union, you can only access members **common to all** variants until you narrow.

## 7.2 Intersection `&` — "all of"

```ts
type Timestamps = { createdAt: Date; updatedAt: Date };
type Post = { id: number; title: string } & Timestamps;

const post: Post = { id: 1, title: "Hi", createdAt: new Date(), updatedAt: new Date() };
```

⚠️ Intersecting incompatible primitives yields `never`: `string & number` → `never`.

## 7.3 Discriminated (tagged) unions ⭐⭐⭐

The most useful pattern in TypeScript: a shared **literal "tag"** property lets the compiler narrow automatically.

```ts
type RequestState<T> =
  | { status: "idle" }
  | { status: "loading" }
  | { status: "success"; data: T }
  | { status: "error"; error: Error };

function render(state: RequestState<string[]>): string {
  switch (state.status) {
    case "idle":
      return "Start searching";
    case "loading":
      return "Loading…";
    case "success":
      return state.data.join(", ");     // data available only here
    case "error":
      return state.error.message;       // error available only here
  }
}
```

**Why it's better than optional fields:**

```ts
// ❌ allows impossible states: loading AND error AND data at once
type BadState = { loading: boolean; data?: string[]; error?: Error };
```

"**Make impossible states unrepresentable.**"

---

# 8. Narrowing & Type Guards ⭐

**Narrowing** = TypeScript refines a broad type to a more specific one inside a code block, based on runtime checks (control-flow analysis).

## 8.1 Built-in narrowing

```ts
function format(value: string | number | Date | null | string[]) {
  if (value === null) return "—";                       // equality
  if (typeof value === "string") return value.trim();   // typeof
  if (typeof value === "number") return value.toFixed(2);
  if (value instanceof Date) return value.toISOString(); // instanceof
  if (Array.isArray(value)) return value.join(", ");     // Array.isArray
  return value;                                          // never here
}

type Fish = { swim(): void };
type Bird = { fly(): void };
function move(animal: Fish | Bird) {
  if ("swim" in animal) animal.swim();                   // `in` operator
  else animal.fly();
}

function greet(name?: string) {
  if (name) return `Hi ${name}`;                         // truthiness (⚠️ also excludes "")
  return "Hi stranger";
}
```

⚠️ `typeof null === "object"` — check `null` first when using `typeof x === "object"`.

## 8.2 User-defined type guards (type predicates) ⭐

```ts
type User = { id: number; name: string };

function isUser(value: unknown): value is User {
  return (
    typeof value === "object" &&
    value !== null &&
    "id" in value && typeof value.id === "number" &&
    "name" in value && typeof value.name === "string"
  );
}

const data: unknown = JSON.parse('{"id":1,"name":"Asha"}');
if (isUser(data)) {
  console.log(data.name);   // data: User
}

// Filtering with predicates (TS 5.5+ infers simple predicates automatically)
const values = [1, null, 2, undefined, 3];
const nums = values.filter((v) => v != null);   // number[] ✅ (inferred predicate)
```

## 8.3 Assertion functions

```ts
function assert(condition: unknown, msg: string): asserts condition {
  if (!condition) throw new Error(msg);
}

function process(input: string | null) {
  assert(input !== null, "input required");
  input.toUpperCase();   // input: string after the assertion
}
```

## 8.4 Exhaustiveness checking with `never` ⭐

```ts
type Shape =
  | { kind: "circle"; radius: number }
  | { kind: "square"; side: number }
  | { kind: "rect"; w: number; h: number };

function assertNever(x: never): never {
  throw new Error(`Unhandled case: ${JSON.stringify(x)}`);
}

function area(shape: Shape): number {
  switch (shape.kind) {
    case "circle":
      return Math.PI * shape.radius ** 2;
    case "square":
      return shape.side ** 2;
    case "rect":
      return shape.w * shape.h;
    default:
      return assertNever(shape);   // if a new kind is added and not handled → compile error here
  }
}
```

## 8.5 Non-null assertion `!` ⚠️

```ts
const el = document.getElementById("app")!;   // "trust me, it's not null"
```

Use sparingly — it's an unchecked promise to the compiler. Prefer a check or an assertion function.

## 8.6 Optional chaining & nullish coalescing

```ts
type Profile = { address?: { city?: string } };
function city(p?: Profile) {
  return p?.address?.city ?? "Unknown";   // string
}
```

---

# 9. Generics ⭐

**Generics** let you write reusable code that works with **many types** while keeping type information — "type parameters" are to types what function parameters are to values.

## 9.1 Generic functions

```ts
function identity<T>(value: T): T {
  return value;
}
const a = identity("hello");      // T inferred as string
const b = identity<number>(42);   // explicit

function first<T>(arr: T[]): T | undefined {
  return arr[0];
}
const f = first([1, 2, 3]);       // number | undefined
```

Without generics you'd lose type info (`any`) or duplicate code per type.

## 9.2 Constraints with `extends` ⭐

```ts
function longest<T extends { length: number }>(a: T, b: T): T {
  return a.length >= b.length ? a : b;
}
longest("abc", "de");     // string
longest([1, 2], [3]);     // number[]
longest(10, 20);          // ❌ number doesn't have 'length'
```

## 9.3 `keyof` constraints — type-safe property access ⭐

```ts
function getProp<T, K extends keyof T>(obj: T, key: K): T[K] {
  return obj[key];
}

const user = { id: 1, name: "Asha", active: true };
const userName = getProp(user, "name");   // string
const active = getProp(user, "active");   // boolean
getProp(user, "email");                   // ❌ '"email"' is not assignable to '"id" | "name" | "active"'
```

## 9.4 Multiple type parameters & defaults

```ts
function mapValues<T, U>(arr: T[], fn: (item: T) => U): U[] {
  return arr.map(fn);
}
const lengths = mapValues(["a", "bb"], (s) => s.length);   // number[]

type ApiResponse<TData = unknown, TError = string> = {
  data?: TData;
  error?: TError;
};
const res: ApiResponse<{ id: number }> = { data: { id: 1 } };
```

## 9.5 Generic types, interfaces & classes

```ts
type Box<T> = { value: T };

interface Repository<T extends { id: string }> {
  findById(id: string): Promise<T | undefined>;
  save(entity: T): Promise<void>;
}

class Stack<T> {
  #items: T[] = [];
  push(item: T): void {
    this.#items.push(item);
  }
  pop(): T | undefined {
    return this.#items.pop();
  }
  get size(): number {
    return this.#items.length;
  }
}

const stack = new Stack<number>();
stack.push(1);
stack.push("2");   // ❌ string is not assignable to number
```

## 9.6 `const` type parameters (TS 5.0)

```ts
function routes<const T extends readonly string[]>(paths: T): T {
  return paths;
}
const r = routes(["/home", "/about"]);   // readonly ["/home", "/about"] — literal types kept
```

## 9.7 `NoInfer` (TS 5.4)

```ts
function createState<T extends string>(states: T[], initial: NoInfer<T>) {
  return { states, current: initial };
}
createState(["idle", "loading"], "idle");
createState(["idle", "loading"], "done");   // ❌ "done" not in the union (not used to infer T)
```

## 9.8 Generic tips 💡

- Name meaningfully when it helps: `TItem`, `TKey`, `TResponse`.
- If a type parameter appears **only once**, you probably don't need it: `function log<T>(x: T): void` → `function log(x: unknown): void`.
- Let callers **infer** — design so explicit type arguments are rarely needed.
- Don't over-constrain; don't under-constrain.

---

# 10. Type Operators & Type-Level Programming ⭐

## 10.1 `keyof`

```ts
type User = { id: number; name: string; email: string };
type UserKey = keyof User;   // "id" | "name" | "email"
```

## 10.2 `typeof` (type context)

```ts
const defaultConfig = { port: 3000, host: "localhost", debug: false };
type Config = typeof defaultConfig;   // { port: number; host: string; debug: boolean }

function createServer(opts: Partial<Config>) { return { ...defaultConfig, ...opts }; }
```

## 10.3 Indexed access types

```ts
type User = { id: number; profile: { city: string; tags: string[] } };
type UserId = User["id"];                 // number
type City = User["profile"]["city"];      // string
type Tag = User["profile"]["tags"][number]; // string (element type of an array)

const COLORS = ["red", "green"] as const;
type Color = (typeof COLORS)[number];     // "red" | "green"
```

## 10.4 Conditional types

```ts
type IsString<T> = T extends string ? true : false;
type A = IsString<"hi">;   // true
type B = IsString<42>;     // false

// Distributive over unions (when T is a naked type parameter)
type ToArray<T> = T extends unknown ? T[] : never;
type C = ToArray<string | number>;   // string[] | number[]

// Prevent distribution with brackets
type ToArrayNonDist<T> = [T] extends [unknown] ? T[] : never;
type D = ToArrayNonDist<string | number>;   // (string | number)[]
```

## 10.5 `infer` — extract types inside conditional types

```ts
type ElementType<T> = T extends (infer U)[] ? U : never;
type E = ElementType<string[]>;            // string

type MyReturnType<T> = T extends (...args: any[]) => infer R ? R : never;
type R = MyReturnType<() => Promise<number>>;   // Promise<number>

type UnwrapPromise<T> = T extends Promise<infer V> ? UnwrapPromise<V> : T;
type V = UnwrapPromise<Promise<Promise<string>>>; // string

type FirstArg<F> = F extends (first: infer A, ...rest: any[]) => any ? A : never;
type FA = FirstArg<(id: number, name: string) => void>; // number
```

## 10.6 Mapped types ⭐

Transform every property of a type.

```ts
type User = { id: number; name: string; email: string };

type Optional<T> = { [K in keyof T]?: T[K] };          // like Partial
type ReadOnly<T> = { readonly [K in keyof T]: T[K] };  // like Readonly
type Mutable<T> = { -readonly [K in keyof T]: T[K] };  // remove readonly
type Concrete<T> = { [K in keyof T]-?: T[K] };         // remove optional

type Nullable<T> = { [K in keyof T]: T[K] | null };
type UserFlags = { [K in keyof User]: boolean };        // { id: boolean; name: boolean; email: boolean }
```

### Key remapping with `as` (TS 4.1)

```ts
type Getters<T> = {
  [K in keyof T as `get${Capitalize<string & K>}`]: () => T[K];
};
type UserGetters = Getters<{ name: string; age: number }>;
// { getName: () => string; getAge: () => number }

// Filter keys: keep only string-valued properties
type StringKeys<T> = { [K in keyof T as T[K] extends string ? K : never]: T[K] };
type S = StringKeys<{ id: number; name: string; email: string }>;   // { name: string; email: string }
```

## 10.7 Template literal types

```ts
type HttpMethod = "GET" | "POST";
type Endpoint = "/users" | "/posts";
type Route = `${HttpMethod} ${Endpoint}`;
// "GET /users" | "GET /posts" | "POST /users" | "POST /posts"

type EventName<T extends string> = `on${Capitalize<T>}`;
type ClickEvent = EventName<"click">;   // "onClick"

type CssSize = `${number}px` | `${number}rem`;
const size: CssSize = "12px";
const badSize: CssSize = "12em";        // ❌

// Intrinsic string types: Uppercase, Lowercase, Capitalize, Uncapitalize
type Shout = Uppercase<"hello">;        // "HELLO"
```

## 10.8 Recursive types

```ts
type Json = string | number | boolean | null | Json[] | { [key: string]: Json };

type DeepPartial<T> = {
  [K in keyof T]?: T[K] extends object ? DeepPartial<T[K]> : T[K];
};

type DeepReadonly<T> = {
  readonly [K in keyof T]: T[K] extends object ? DeepReadonly<T[K]> : T[K];
};

type Settings = { theme: { color: string; font: { size: number } } };
const patch: DeepPartial<Settings> = { theme: { font: { size: 14 } } };
```

---

# 11. Utility Types ⭐

Built-in generic types for common transformations.

| Utility | Result | Example |
| --- | --- | --- |
| `Partial<T>` | All properties optional | Update payloads |
| `Required<T>` | All properties required | Resolved config |
| `Readonly<T>` | All properties readonly | Immutable state |
| `Pick<T, K>` | Only keys `K` | `Pick<User, "id" \| "name">` |
| `Omit<T, K>` | All except `K` | `Omit<User, "password">` |
| `Record<K, V>` | Object with keys `K`, values `V` | `Record<Status, string>` |
| `Exclude<U, E>` | Remove members of union `U` assignable to `E` | `Exclude<"a" \| "b", "a">` → `"b"` |
| `Extract<U, E>` | Keep members assignable to `E` | `Extract<Shape, { kind: "circle" }>` |
| `NonNullable<T>` | Remove `null \| undefined` | |
| `ReturnType<F>` | Function's return type | `ReturnType<typeof useAuth>` |
| `Parameters<F>` | Tuple of parameter types | `Parameters<typeof fetchUser>[0]` |
| `ConstructorParameters<C>` / `InstanceType<C>` | For classes | |
| `Awaited<T>` | Unwrap Promise (recursively) | `Awaited<ReturnType<typeof getUser>>` |
| `NoInfer<T>` | Block inference from this position | §9.7 |
| `Uppercase` / `Lowercase` / `Capitalize` / `Uncapitalize` | String literal transforms | |

```ts
type User = { id: number; name: string; email: string; password: string; role: "admin" | "user" };

type PublicUser = Omit<User, "password">;
type UserPreview = Pick<User, "id" | "name">;
type UserUpdate = Partial<Omit<User, "id">>;          // id can't be changed; all else optional
type CreateUserInput = Omit<User, "id"> & { confirmPassword: string };
type RoleLabels = Record<User["role"], string>;

const labels: RoleLabels = { admin: "Administrator", user: "Member" };

async function getUser(id: number) {
  return { id, name: "Asha" };
}
type GetUserResult = Awaited<ReturnType<typeof getUser>>;   // { id: number; name: string }
type GetUserArg = Parameters<typeof getUser>[0];            // number
```

### Implement them yourself (interview favourite) ⭐

```ts
type MyPartial<T> = { [K in keyof T]?: T[K] };
type MyRequired<T> = { [K in keyof T]-?: T[K] };
type MyReadonly<T> = { readonly [K in keyof T]: T[K] };
type MyPick<T, K extends keyof T> = { [P in K]: T[P] };
type MyRecord<K extends PropertyKey, V> = { [P in K]: V };
type MyExclude<T, U> = T extends U ? never : T;
type MyExtract<T, U> = T extends U ? T : never;
type MyOmit<T, K extends PropertyKey> = MyPick<T, MyExclude<keyof T, K>>;
type MyNonNullable<T> = T extends null | undefined ? never : T;
type MyReturnType<T extends (...args: any) => any> = T extends (...args: any) => infer R ? R : never;
type MyParameters<T extends (...args: any) => any> = T extends (...args: infer P) => any ? P : never;
type MyAwaited<T> = T extends PromiseLike<infer V> ? MyAwaited<V> : T;
```

⚠️ `Omit` is **not** distributive over unions — `Omit<A | B, "x">` merges A and B's common keys. Use a distributive version for unions:

```ts
type DistributiveOmit<T, K extends PropertyKey> = T extends unknown ? Omit<T, K> : never;
```

---

# 12. Classes

```ts
interface Shape {
  area(): number;
}

abstract class BaseShape implements Shape {
  static count = 0;                         // static member
  constructor(public readonly name: string) {   // parameter property: declares + assigns
    BaseShape.count++;
  }
  abstract area(): number;                  // must be implemented by subclasses
  describe(): string {
    return `${this.name} with area ${this.area().toFixed(2)}`;
  }
}

class Circle extends BaseShape {
  #radius: number;                          // JS private field (runtime-private)
  constructor(radius: number) {
    super("circle");
    this.#radius = radius;
  }
  override area(): number {                 // `override` keyword (noImplicitOverride)
    return Math.PI * this.#radius ** 2;
  }
  get radius(): number {                    // getter
    return this.#radius;
  }
  set radius(value: number) {               // setter
    if (value <= 0) throw new Error("Radius must be positive");
    this.#radius = value;
  }
}

const c = new Circle(2);
c.describe();
c.name = "x";        // ❌ Cannot assign to 'name' because it is a read-only property
new BaseShape("x");  // ❌ Cannot create an instance of an abstract class
```

### Access modifiers

| Modifier | Accessible from |
| --- | --- |
| `public` (default) | Anywhere |
| `protected` | Class and subclasses |
| `private` | Only the class — **compile-time only** (still accessible at runtime via JS) |
| `#field` | Only the class — **truly private at runtime** (JS feature) |
| `readonly` | Assign only in declaration/constructor |

### Definite assignment

```ts
class Service {
  client!: { get(url: string): Promise<unknown> };   // `!` = "I'll initialize it elsewhere" ⚠️
  init(client: Service["client"]) {
    this.client = client;
  }
}
```

### Structural typing applies to classes

```ts
class Point2D { constructor(public x: number, public y: number) {} }
const p: Point2D = { x: 1, y: 2 };   // ✅ same shape — no instanceof relationship needed
```

### Decorators

TS 5.0 supports **standard ECMAScript decorators** (stage 3). Legacy `experimentalDecorators` is still used by Angular, NestJS, TypeORM. Know they exist; usage is framework-specific.

---

# 13. Enums & Alternatives

```ts
enum Direction {
  Up,          // 0
  Down,        // 1
}

enum Status {
  Active = "ACTIVE",
  Inactive = "INACTIVE",
}

const s: Status = Status.Active;
```

### Problems with enums ⚠️

- They **emit runtime code** (an object) — not "erasable"; incompatible with Node's type stripping / `erasableSyntaxOnly`.
- Numeric enums accept any number in older versions and have reverse mappings.
- Nominal-ish behaviour: a string `"ACTIVE"` isn't assignable to `Status`.
- `const enum` has pitfalls with `isolatedModules` and libraries.

### Preferred alternatives ⭐

```ts
// 1. Union of string literals (zero runtime cost)
type Status = "active" | "inactive";

// 2. `as const` object when you also need runtime values (iteration, lookup)
const Status = {
  Active: "active",
  Inactive: "inactive",
} as const;
type StatusValue = (typeof Status)[keyof typeof Status];   // "active" | "inactive"

function setStatus(s: StatusValue) {}
setStatus(Status.Active);   // ✅
setStatus("inactive");      // ✅ plain strings work too
Object.values(Status);      // ["active", "inactive"] at runtime
```

---

# 14. Modules, Declarations & `.d.ts`

## 14.1 Importing & exporting types

```ts
// types.ts
export type User = { id: number; name: string };
export interface Product { sku: string }
```

```ts
// app.ts
import type { User } from "./types";       // type-only import: erased completely
import { type Product, formatPrice } from "./utils"; // inline type modifier
export type { User };                       // type-only re-export
```

With `verbatimModuleSyntax`, you **must** use `import type` for type-only imports — the compiler then knows exactly which imports to drop.

## 14.2 Declaration files (`.d.ts`)

`.d.ts` files contain **only types** — describing JavaScript code that exists elsewhere.

- Libraries ship them (`"types": "dist/index.d.ts"` in package.json) or the community provides them in **DefinitelyTyped**: `npm i -D @types/lodash`.
- `lib.dom.d.ts`, `lib.es2023.d.ts` describe built-ins (selected via `lib`).
- Generate yours with `"declaration": true`.

## 14.3 `declare` — describe things that exist at runtime

```ts
// globals.d.ts
declare const __APP_VERSION__: string;          // injected by the bundler
declare function gtag(...args: unknown[]): void; // third-party script global

// Typing a JS module with no types
declare module "legacy-lib" {
  export function doThing(input: string): number;
}

// Non-code imports (bundlers)
declare module "*.svg" {
  const src: string;
  export default src;
}
declare module "*.module.css" {
  const classes: Record<string, string>;
  export default classes;
}
```

## 14.4 Module augmentation & global augmentation

```ts
// Add a property to Express's Request
import "express";
declare module "express-serve-static-core" {
  interface Request {
    user?: { id: string; role: "admin" | "user" };
  }
}

// Add to the global Window
declare global {
  interface Window {
    __INITIAL_STATE__: unknown;
  }
}
export {};
```

## 14.5 Environment variables typing

```ts
// vite-env.d.ts
interface ImportMetaEnv {
  readonly VITE_API_URL: string;
}
interface ImportMeta {
  readonly env: ImportMetaEnv;
}
```

## 14.6 Namespaces

`namespace` is a legacy TS-only way of grouping code (emits runtime code). Use **ES modules** instead. You'll still see `declare namespace` in `.d.ts` files describing globals.

---

# 15. Type Compatibility: any, unknown, never & Variance ⭐

## 15.1 Structural typing ⭐

Compatibility is based on **shape**, not on names or declarations.

```ts
type Person = { name: string };
type Employee = { name: string; company: string };

const emp: Employee = { name: "Asha", company: "Acme" };
const person: Person = emp;          // ✅ Employee has everything Person needs (extra props OK)
const e2: Employee = person;         // ❌ Property 'company' is missing
```

A type with **more** properties is a **subtype** of one with fewer.

## 15.2 The type hierarchy

```text
               unknown            ← top type: everything is assignable TO it
          /      |       \
     string   number   object …
        |        |        |
     "a"      42       {x:1}
          \      |       /
               never              ← bottom type: assignable to everything; nothing (but never) assignable to it

     any = escape hatch: assignable to and from everything (disables checking)
```

| | Can assign anything **to** it? | Can assign it **to** anything? | Can use it without checks? |
| --- | --- | --- | --- |
| `any` | ✅ | ✅ | ✅ ⚠️ |
| `unknown` | ✅ | ❌ (only to `unknown`/`any`) | ❌ must narrow |
| `never` | ❌ (only `never`) | ✅ | — (no values exist) |

## 15.3 Widening & narrowing of literals

```ts
let a = "hello";            // string (widened — let can change)
const b = "hello";          // "hello"
const obj = { kind: "x" };  // { kind: string } — properties widen
const obj2 = { kind: "x" } as const; // { readonly kind: "x" }
```

This matters for discriminated unions — a widened `kind: string` won't match `"circle" | "square"`.

## 15.4 Variance (advanced, interviews) ⭐

How subtyping of parts relates to subtyping of the whole.

| | Meaning | Example |
| --- | --- | --- |
| **Covariant** | Preserves direction: `Dog ⊂ Animal` ⇒ `F<Dog> ⊂ F<Animal>` | Return types, readonly arrays |
| **Contravariant** | Reverses direction: `F<Animal> ⊂ F<Dog>` | Function **parameters** (with `strictFunctionTypes`) |
| **Invariant** | Neither | Mutable containers (in theory) |
| **Bivariant** | Both (unsound) | Method parameters (method shorthand syntax) |

```ts
type Animal = { name: string };
type Dog = { name: string; bark(): void };

let handleAnimal = (a: Animal) => {};
let handleDog = (d: Dog) => {};

handleDog = handleAnimal;   // ✅ a function accepting any Animal can handle a Dog
handleAnimal = handleDog;   // ❌ a Dog-handler might call bark() on a Cat
```

⚠️ TS arrays are covariant even when mutable (`Dog[]` assignable to `Animal[]`) — a known unsoundness for convenience.

Variance annotations for generics (rarely needed): `type Box<in out T>`, `interface Producer<out T>`, `interface Consumer<in T>`.

---

# 16. Async Code & Error Handling

```ts
type User = { id: number; name: string };

async function fetchUser(id: number): Promise<User> {
  const res = await fetch(`/api/users/${id}`);
  if (!res.ok) throw new Error(`HTTP ${res.status}`);
  return (await res.json()) as User;   // ⚠️ cast — validate in real code (§17)
}

// Parallel with tuple types preserved
async function load() {
  const [user, count] = await Promise.all([fetchUser(1), Promise.resolve(42)]);
  // user: User, count: number
}
```

### `catch` variables are `unknown` ⭐

```ts
try {
  JSON.parse("{bad");
} catch (error) {
  // error: unknown (useUnknownInCatchVariables, part of strict)
  console.log(error.message);   // ❌ 'error' is of type 'unknown'
  if (error instanceof Error) console.log(error.message);   // ✅
}

function getErrorMessage(error: unknown): string {
  if (error instanceof Error) return error.message;
  if (typeof error === "string") return error;
  return "Unknown error";
}
```

### Result type (errors as values) 💡

```ts
type Result<T, E = Error> = { ok: true; value: T } | { ok: false; error: E };

async function safeFetch<T>(url: string): Promise<Result<T>> {
  try {
    const res = await fetch(url);
    if (!res.ok) return { ok: false, error: new Error(`HTTP ${res.status}`) };
    return { ok: true, value: (await res.json()) as T };
  } catch (e) {
    return { ok: false, error: e instanceof Error ? e : new Error(String(e)) };
  }
}

async function demo() {
  const result = await safeFetch<{ id: number }>("/api/x");
  if (result.ok) console.log(result.value.id);
  else console.error(result.error.message);
}
```

### Custom error classes

```ts
class HttpError extends Error {
  constructor(public readonly status: number, message: string) {
    super(message);
    this.name = "HttpError";
  }
}

function handle(e: unknown) {
  if (e instanceof HttpError && e.status === 401) {
    // redirect to login
  }
}
```

---

# 17. Runtime Validation & Branded Types ⭐

## 17.1 Types disappear at runtime ⚠️

```ts
type User = { id: number; name: string };
const user = JSON.parse('{"id":"oops"}') as User;   // compiles fine…
// …but user.id is a string and user.name is undefined at runtime
```

Data from **APIs, `JSON.parse`, `localStorage`, forms, env vars, URL params** is untrusted. TypeScript can't verify it — **validate at the boundary**.

## 17.2 Schema validation with Zod (types from schemas) ⭐

```ts
import { z } from "zod";

const UserSchema = z.object({
  id: z.number().int().positive(),
  name: z.string().min(1),
  email: z.string().email(),
  role: z.enum(["admin", "user"]).default("user"),
  tags: z.array(z.string()).optional(),
});

type User = z.infer<typeof UserSchema>;   // TS type derived from the schema — single source of truth

async function getUser(id: number): Promise<User> {
  const res = await fetch(`/api/users/${id}`);
  return UserSchema.parse(await res.json());   // throws if invalid
}

const parsed = UserSchema.safeParse({ id: 1 });
if (!parsed.success) console.log(parsed.error.issues);

// Env vars
const Env = z.object({
  DATABASE_URL: z.string().url(),
  PORT: z.coerce.number().default(3000),
});
export const env = Env.parse(process.env);
```

Alternatives: Valibot (smaller), ArkType, Yup, io-ts, TypeBox (JSON Schema).

## 17.3 Branded (nominal) types ⭐

Structural typing means `UserId` and `OrderId` (both `string`) are interchangeable — a source of bugs. **Brand** them:

```ts
type Brand<T, B extends string> = T & { readonly __brand: B };

type UserId = Brand<string, "UserId">;
type OrderId = Brand<string, "OrderId">;
type Email = Brand<string, "Email">;

const toUserId = (id: string) => id as UserId;
function toEmail(value: string): Email {
  if (!/^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(value)) throw new Error("Invalid email");
  return value as Email;   // only obtainable through validation
}

function getOrder(id: OrderId) { return id; }

const uid = toUserId("u_123");
getOrder(uid);             // ❌ 'UserId' is not assignable to 'OrderId'
getOrder("o_1");           // ❌ plain string not allowed either
```

Use for IDs, validated strings (emails, URLs), units (`Meters`, `Seconds`), sanitized HTML.

---

# 18. TypeScript with React & Node

## 18.1 React essentials

```tsx
import { useState, type ReactNode, type ComponentPropsWithoutRef } from "react";

type ButtonProps = ComponentPropsWithoutRef<"button"> & {
  variant?: "primary" | "ghost";
  icon?: ReactNode;
};

export function Button({ variant = "primary", icon, children, ...rest }: ButtonProps) {
  return (
    <button className={`btn-${variant}`} {...rest}>
      {icon}
      {children}
    </button>
  );
}

function Counter() {
  const [user, setUser] = useState<{ name: string } | null>(null);
  const onChange = (e: React.ChangeEvent<HTMLInputElement>) => setUser({ name: e.target.value });
  return <input onChange={onChange} value={user?.name ?? ""} />;
}
```

Full React + TypeScript coverage (events, hooks, context, reducers, generic & polymorphic components): see React notes `23-typescript-with-react.md`.

## 18.2 Node / Express essentials

```ts
import express, { type Request, type Response, type NextFunction } from "express";

const app = express();
app.use(express.json());

type CreateUserBody = { name: string; email: string };
type Params = { id: string };

app.get("/users/:id", (req: Request<Params>, res: Response) => {
  res.json({ id: req.params.id });
});

app.post("/users", (req: Request<{}, unknown, CreateUserBody>, res: Response) => {
  const { name, email } = req.body;   // typed (but still validate at runtime!)
  res.status(201).json({ name, email });
});

app.use((err: unknown, _req: Request, res: Response, _next: NextFunction) => {
  res.status(500).json({ message: err instanceof Error ? err.message : "Internal error" });
});
```

Node built-ins types: `npm i -D @types/node`. End-to-end type safety across client and server: **tRPC**, **GraphQL Codegen**, **OpenAPI → `openapi-typescript`**, Hono RPC.

---

# 19. Common Errors & Fixes ⭐

| Code | Message (short) | Typical cause | Fix |
| --- | --- | --- | --- |
| **TS2322** | Type 'X' is not assignable to type 'Y' | Wrong value type | Fix the value or widen the declared type |
| **TS2345** | Argument of type 'X' is not assignable to parameter of type 'Y' | Wrong argument | Convert/validate/narrow before calling |
| **TS2339** | Property 'x' does not exist on type 'Y' | Typo, missing property, union not narrowed | Fix typo; extend type; narrow first |
| **TS2532 / TS18048** | Object is possibly 'undefined' / 'x' is possibly 'undefined' | Optional value used directly | `?.`, `??`, `if (x)`, default values |
| **TS2531 / TS18047** | Object is possibly 'null' | `getElementById`, `querySelector` | Null check; `!` only if guaranteed |
| **TS7006** | Parameter 'x' implicitly has an 'any' type | Missing annotation | Annotate parameters |
| **TS7053** | Element implicitly has 'any' type because expression can't be used to index | `obj[key]` with `key: string` | Type key as `keyof typeof obj`, or use `Record<string, T>` |
| **TS2741** | Property 'x' is missing in type… | Missing required property | Add it or make it optional |
| **TS2353** | Object literal may only specify known properties | Excess property / typo | Remove or add to type |
| **TS2554** | Expected N arguments, but got M | Wrong call arity | Fix call / make param optional |
| **TS2769** | No overload matches this call | None of the overloads fit | Read each overload's error; check argument types |
| **TS18046** | 'x' is of type 'unknown' | Using `unknown` without narrowing | Narrow with `typeof`/`instanceof`/guard/schema |
| **TS2367** | This comparison appears to be unintentional (no overlap) | Comparing incompatible literals | Fix the logic / types |
| **TS2564** | Property has no initializer and is not definitely assigned | Class field not set in constructor | Initialize, make optional, or `!` |
| **TS1484** | 'X' is a type and must be imported using a type-only import | `verbatimModuleSyntax` | `import type { X }` |
| **TS2307** | Cannot find module 'x' or its type declarations | Missing package or `@types/x`; path alias | Install `@types/x`, add `declare module`, fix `paths` |
| **TS2589** | Type instantiation is excessively deep | Overly recursive types | Simplify; add depth limits |
| **TS2304** | Cannot find name 'x' | Missing import, missing `lib`/`types` (e.g. `process` without `@types/node`) | Import it; add `"types": ["node"]` or `lib` |

### Fixing TS7053 (indexing with a string)

```ts
const prices = { apple: 1, mango: 2 };

function priceOf(fruit: string) {
  return prices[fruit];                                   // ❌ TS7053
}
function priceOfFixed(fruit: keyof typeof prices) {
  return prices[fruit];                                   // ✅
}
const prices2: Record<string, number> = { apple: 1 };
function priceOf2(fruit: string) {
  return prices2[fruit];                                  // ✅ number | undefined (with noUncheckedIndexedAccess)
}
```

### Debugging types 💡

- Hover in the editor to see inferred types.
- Force display of a computed type: `type Debug<T> = { [K in keyof T]: T[K] } & {}` ("prettify").
- Ask the compiler: `const _check: ExpectedType = value;`
- Type tests in libraries: `expectTypeOf` (Vitest), `tsd`, `// @ts-expect-error` lines that must error.
- `// @ts-expect-error` > `// @ts-ignore` (the former fails when the error disappears).

---

# 20. Best Practices ⭐

1. **Enable `strict`** (default in TS 6+) plus `noUncheckedIndexedAccess`.
2. **Avoid `any`** — use `unknown` and narrow; ban it with ESLint (`@typescript-eslint/no-explicit-any`).
3. **Let inference work** inside functions; **annotate boundaries** (params, exported returns, public APIs).
4. **Model data with discriminated unions** — make impossible states unrepresentable.
5. **Prefer unions of literals / `as const` objects over enums.**
6. **Validate external data at runtime** (Zod) and derive types from schemas.
7. **Avoid type assertions (`as`)** and non-null assertions (`!`) — each is an unchecked claim.
8. **Use `satisfies`** for config objects to check without losing precise types.
9. **Use `readonly`** for data that shouldn't change (props, state, config).
10. **Keep types close to usage**; share domain types in a `types/` or per-feature module.
11. **Don't over-engineer** type-level code in application code — readability first.
12. **Use `import type`** (`verbatimModuleSyntax`) for clean, predictable emit.
13. **Exhaustive switches** with `never`.
14. **Type-check in CI** (`tsc --noEmit`) since bundlers don't.
15. **Generate types** from sources of truth (OpenAPI, GraphQL schema, database via Prisma/Drizzle).
16. **Use typescript-eslint** with type-aware rules (`no-floating-promises`, `no-misused-promises`, `strict-boolean-expressions`).
17. **Migrate JS gradually**: `allowJs` + `checkJs`, convert file by file, start lenient and tighten.

---

# 21. Real-World Recipes ⭐

### 1. Type a fetch wrapper

```ts
async function http<T>(url: string, init?: RequestInit): Promise<T> {
  const res = await fetch(url, init);
  if (!res.ok) throw new Error(`HTTP ${res.status}`);
  return res.json() as Promise<T>;
}
async function demo() {
  const posts = await http<{ id: number; title: string }[]>("/api/posts");
  posts[0]?.title;
}
```

### 2. Get keys/values of an object as types

```ts
const ROUTES = { home: "/", profile: "/profile", settings: "/settings" } as const;
type RouteName = keyof typeof ROUTES;            // "home" | "profile" | "settings"
type RoutePath = (typeof ROUTES)[RouteName];     // "/" | "/profile" | "/settings"
```

### 3. Type-safe `Object.keys` / `Object.entries`

```ts
const typedKeys = <T extends object>(obj: T) => Object.keys(obj) as (keyof T)[];
const typedEntries = <T extends object>(obj: T) =>
  Object.entries(obj) as { [K in keyof T]: [K, T[K]] }[keyof T][];
```

(`Object.keys` returns `string[]` by design, because objects may have extra keys at runtime.)

### 4. Make some properties optional / required

```ts
type PartialBy<T, K extends keyof T> = Omit<T, K> & Partial<Pick<T, K>>;
type RequiredBy<T, K extends keyof T> = Omit<T, K> & Required<Pick<T, K>>;

type User = { id: number; name: string; email?: string };
type NewUser = PartialBy<User, "id">;               // id optional
type UserWithEmail = RequiredBy<User, "email">;     // email required
```

### 5. Prettify an intersection for readable hovers

```ts
type Prettify<T> = { [K in keyof T]: T[K] } & {};
type Combined = Prettify<{ a: number } & { b: string }>;   // shows { a: number; b: string }
```

### 6. Type-safe event emitter

```ts
type Events = {
  login: { userId: string };
  logout: undefined;
  error: { message: string };
};

class Emitter<E extends Record<string, unknown>> {
  #handlers: { [K in keyof E]?: Array<(payload: E[K]) => void> } = {};

  on<K extends keyof E>(event: K, handler: (payload: E[K]) => void): () => void {
    (this.#handlers[event] ??= []).push(handler);
    return () => {
      this.#handlers[event] = this.#handlers[event]?.filter((h) => h !== handler);
    };
  }

  emit<K extends keyof E>(event: K, payload: E[K]): void {
    this.#handlers[event]?.forEach((h) => h(payload));
  }
}

const bus = new Emitter<Events>();
bus.on("login", (p) => console.log(p.userId));   // p: { userId: string }
bus.emit("login", { userId: "u1" });
bus.emit("login", { id: "u1" });                 // ❌ wrong payload
```

### 7. Exhaustive mapping with `Record`

```ts
type Status = "idle" | "loading" | "success" | "error";
const statusColor: Record<Status, string> = {
  idle: "gray",
  loading: "blue",
  success: "green",
  error: "red",
};   // adding a new Status forces you to add a color here
```

### 8. Readonly deep config with `as const` + `satisfies`

```ts
type FeatureFlags = Record<string, { enabled: boolean; rollout?: number }>;

const flags = {
  newCheckout: { enabled: true, rollout: 50 },
  darkMode: { enabled: false },
} as const satisfies FeatureFlags;

type FlagName = keyof typeof flags;   // "newCheckout" | "darkMode"
```

### 9. Dot-notation paths of an object (type-level)

```ts
type Paths<T> = T extends object
  ? { [K in keyof T & string]: T[K] extends object ? K | `${K}.${Paths<T[K]>}` : K }[keyof T & string]
  : never;

type Settings = { theme: { color: string; font: { size: number } }; lang: string };
type SettingPath = Paths<Settings>;
// "theme" | "theme.color" | "theme.font" | "theme.font.size" | "lang"
```

Used by form libraries (React Hook Form's `register("address.city")`) and i18n keys.

### 10. Function that accepts only specific keys of a certain value type

```ts
type KeysOfType<T, V> = { [K in keyof T]-?: T[K] extends V ? K : never }[keyof T];

type Product = { name: string; price: number; stock: number; sku: string };
function sortByNumber(items: Product[], key: KeysOfType<Product, number>) {
  return [...items].sort((a, b) => a[key] - b[key]);
}
sortByNumber([], "price");   // ✅
sortByNumber([], "name");    // ❌ name is a string
```

### 11. Discriminated union state machine reducer

```ts
type State =
  | { status: "idle" }
  | { status: "loading"; requestId: string }
  | { status: "success"; data: string[] }
  | { status: "error"; message: string };

type Action =
  | { type: "fetch"; requestId: string }
  | { type: "resolve"; data: string[] }
  | { type: "reject"; message: string };

function reducer(state: State, action: Action): State {
  switch (action.type) {
    case "fetch":
      return { status: "loading", requestId: action.requestId };
    case "resolve":
      return state.status === "loading" ? { status: "success", data: action.data } : state;
    case "reject":
      return state.status === "loading" ? { status: "error", message: action.message } : state;
  }
}
```

### 12. Typing a debounce utility generically

```ts
function debounce<A extends unknown[]>(fn: (...args: A) => void, ms: number) {
  let timer: ReturnType<typeof setTimeout> | undefined;
  return (...args: A) => {
    clearTimeout(timer);
    timer = setTimeout(() => fn(...args), ms);
  };
}
const search = debounce((query: string, page: number) => {}, 300);
search("react", 1);
search(1);   // ❌ arguments checked against the original function
```

---

# 22. Cheat Sheet ⭐

### Basics

```ts
let s: string; let n: number; let b: boolean; let u: unknown; let v: void; let nv: never;
let arr: number[]; let ro: readonly string[]; let tup: [string, number];
let lit: "a" | "b"; let opt: { a?: string; readonly b: number };
let fn: (x: number) => string;
let rec: Record<string, number>; let idx: { [key: string]: number };
```

### Declarations

| Syntax | Meaning |
| --- | --- |
| `type T = …` | Type alias (anything) |
| `interface I { … }` | Object contract (mergeable) |
| `A \| B` / `A & B` | Union / intersection |
| `x as T` | Type assertion ⚠️ |
| `x!` | Non-null assertion ⚠️ |
| `value satisfies T` | Check without widening |
| `as const` | Deep readonly literal types |
| `x is T` | Type predicate |
| `asserts x is T` | Assertion function |
| `import type { T }` | Type-only import |
| `declare …` | Describe existing runtime values |

### Type operators

| Syntax | Meaning |
| --- | --- |
| `keyof T` | Union of keys |
| `typeof value` | Type of a value |
| `T[K]`, `T[number]` | Indexed access / element type |
| `T extends U ? X : Y` | Conditional type |
| `infer R` | Capture a type in a conditional |
| `{ [K in keyof T]: … }` | Mapped type |
| `{ [K in keyof T as …]: … }` | Key remapping |
| `-readonly`, `-?` | Remove modifiers |
| `` `prefix${T}` `` | Template literal type |
| `<T extends C = D>` | Generic with constraint & default |
| `<const T>` | Const type parameter |

### Utility types

`Partial` · `Required` · `Readonly` · `Pick` · `Omit` · `Record` · `Exclude` · `Extract` · `NonNullable` · `ReturnType` · `Parameters` · `ConstructorParameters` · `InstanceType` · `Awaited` · `NoInfer` · `Uppercase` · `Lowercase` · `Capitalize` · `Uncapitalize`

### Narrowing

`typeof x === "string"` · `x instanceof Date` · `"key" in obj` · `Array.isArray(x)` · `x === null` · truthiness · discriminant `switch (x.kind)` · custom `x is T` guards · `assertNever(x)`

### CLI

| Command | Purpose |
| --- | --- |
| `npx tsc --init` | Create tsconfig |
| `npx tsc --noEmit` | Type-check only |
| `npx tsc -w` | Watch |
| `npx tsc -b` | Build project references |
| `npx tsc --showConfig` | Print resolved config |
| `npx tsc --traceResolution` | Debug module resolution |
| `npx tsc --listFiles` | Files included in compilation |
| `npx tsc --generateTrace dir` | Performance tracing |

---

# 23. Interview Questions & Answers ⭐

### Fundamentals

**1. What is TypeScript and why use it?**
> "A statically typed superset of JavaScript that compiles to plain JavaScript. It catches type errors at compile time, gives excellent editor tooling — autocomplete, safe refactoring — and makes large codebases easier to maintain because function signatures and data shapes are explicit."

**2. Does TypeScript exist at runtime?**
> "No. Types are erased during compilation; the output is plain JavaScript. So TypeScript can't validate runtime data like API responses — that needs runtime validation such as Zod. A few features — enums, namespaces, parameter properties, decorators — do emit code."

**3. What is structural typing?**
> "Compatibility is based on the shape of types, not their names. If an object has all the properties a type requires, it's assignable to that type, even if it was declared separately or has extra properties. That's different from nominal typing in Java or C#. To get nominal behaviour I use branded types."

**4. `type` vs `interface`?**
> "Both describe object shapes. Interfaces support declaration merging and `extends`, which is useful for public APIs and augmenting library types. Type aliases can express everything else — unions, tuples, primitives, mapped and conditional types — and use `&` to combine. I default to `type` and use `interface` when I need merging or an extendable public contract; consistency in the codebase matters most."

**5. `any` vs `unknown` vs `never`?**
> "`any` disables type checking — you can do anything with it, which is unsafe. `unknown` accepts any value but you must narrow it before using it, so it's the safe choice for untrusted data. `never` represents values that can't exist — functions that always throw, exhaustive checks, or impossible branches in conditional types."

**6. What is type inference?**
> "TypeScript automatically deduces types from values, return statements and context, like callback parameters. I let it infer local variables and annotate function parameters, exported return types and empty collections."

**7. What does `strict` mode do?**
> "It enables a family of checks: `strictNullChecks` (the most important — null and undefined must be handled), `noImplicitAny`, `strictFunctionTypes`, `strictPropertyInitialization`, `strictBindCallApply`, `useUnknownInCatchVariables`, `noImplicitThis` and `alwaysStrict`. Since TypeScript 6 it's on by default."

**8. `void` vs `undefined` vs `never` as return types?**
> "`void` means the caller shouldn't use the return value. `undefined` means the function explicitly returns undefined. `never` means it never returns normally — it throws or loops forever."

### Types

**9. What are union and intersection types?**
> "A union `A | B` is a value of either type — you can only use common members until you narrow. An intersection `A & B` combines both types' members into one."

**10. What is a discriminated union?**
> "A union of object types sharing a literal property like `status` or `kind`. Checking that property narrows the union automatically, which is perfect for modelling states — idle, loading, success with data, error with error — and makes impossible states unrepresentable. Combined with a `never` check, the compiler enforces exhaustive handling."

**11. What is narrowing? Name some type guards.**
> "Refining a broad type to a more specific one based on runtime checks the compiler understands: `typeof`, `instanceof`, `in`, equality checks, truthiness, `Array.isArray`, discriminant properties, and user-defined type predicates (`x is T`) and assertion functions."

**12. What is a type predicate?**
> "A function return type of the form `param is Type`. When the function returns true, TypeScript narrows the argument to that type in the calling code — useful for validating `unknown` data or filtering arrays."

**13. What are literal types and `as const`?**
> "Literal types are exact values like `'admin'` or `42`. `as const` makes an expression deeply readonly and keeps literal types instead of widening them to `string` or `number` — useful for config objects and deriving unions from arrays."

**14. What does `satisfies` do?**
> "It checks that a value conforms to a type without changing the value's inferred type. So I get validation — missing keys, wrong values — and still keep precise literal types for autocomplete, unlike a type annotation which widens to the declared type."

**15. Type assertion (`as`) vs type annotation?**
> "An annotation declares and checks a type. An assertion tells the compiler to treat a value as a type, overriding its judgement with minimal checking — it's unsafe if wrong. I avoid assertions except at well-understood boundaries."

**16. What is the non-null assertion operator?**
> "`!` after an expression tells the compiler the value isn't null or undefined. It's unchecked, so I prefer explicit checks, optional chaining or assertion functions."

### Generics & advanced

**17. What are generics? Why use them?**
> "Type parameters that let functions, classes and types work with many types while preserving type information — like `function first<T>(arr: T[]): T | undefined`. Without them you'd duplicate code or lose types with `any`."

**18. What are generic constraints?**
> "Restrictions on a type parameter using `extends`, like `T extends { length: number }` or `K extends keyof T`, so the implementation can safely use certain properties."

**19. Explain `keyof` and `typeof` in type positions.**
> "`keyof T` produces a union of T's property names. `typeof value` gets the type of a runtime value. Together, `keyof typeof obj` gives the keys of an object literal as a union."

**20. What are mapped types?**
> "Types that iterate over keys to transform properties: `{ [K in keyof T]?: T[K] }` makes everything optional — that's how `Partial`, `Readonly` and `Record` are built. Key remapping with `as` can rename or filter keys."

**21. What are conditional types and `infer`?**
> "`T extends U ? X : Y` picks a type based on a condition — like a ternary for types. Inside the condition, `infer` captures a part of the type, e.g. `T extends Promise<infer V> ? V : T` unwraps a promise. That's how `ReturnType`, `Parameters` and `Awaited` work. Conditional types distribute over unions when T is a naked type parameter."

**22. What are template literal types?**
> "String-literal types built with template syntax, like `` `on${Capitalize<Event>}` ``, which combine with unions to generate every combination — useful for event names, CSS units and route strings."

**23. Implement `Pick`, `Omit` and `ReturnType`.**
```ts
type MyPick<T, K extends keyof T> = { [P in K]: T[P] };
type MyOmit<T, K extends PropertyKey> = MyPick<T, Exclude<keyof T, K>>;
type MyReturnType<T extends (...args: any) => any> = T extends (...args: any) => infer R ? R : never;
```

**24. What are utility types you use most?**
> "`Partial` for update payloads, `Pick` and `Omit` for derived DTOs, `Record` for maps, `ReturnType` and `Awaited` for inferring results of functions, `Parameters` for wrappers, `NonNullable`, `Exclude` and `Extract` for unions."

**25. What is variance?**
> "How subtyping of components affects subtyping of the whole. Return types are covariant; function parameters are contravariant under `strictFunctionTypes` — a function accepting `Animal` can be used where one accepting `Dog` is expected, not the reverse. Method parameters are bivariant for historical reasons, and arrays are covariant even though mutable, which is a known unsoundness."

**26. What is declaration merging?**
> "Multiple interface declarations with the same name merge into one. It's used to augment library or global types — adding fields to Express's `Request` or to `Window`."

**27. What are `.d.ts` files?**
> "Declaration files containing only type information that describe JavaScript code — shipped by libraries, generated with `declaration: true`, or provided by DefinitelyTyped `@types` packages. `declare module` lets you type untyped modules or asset imports."

**28. What are branded types?**
> "A technique to simulate nominal typing by intersecting a base type with a unique tag, like `string & { __brand: 'UserId' }`, so different IDs or validated strings can't be mixed up accidentally. Values are created only through constructor or validation functions."

**29. Why avoid enums?**
> "They emit runtime code, aren't compatible with type-stripping runtimes and `erasableSyntaxOnly`, numeric enums have loose behaviour and reverse mappings, and string enums are nominal so plain strings aren't assignable. A union of string literals or an `as const` object with a derived union type gives the same safety with zero runtime cost."

**30. What does `private` vs `#private` mean?**
> "`private` is enforced only by the compiler — the property is still accessible at runtime. `#field` is a JavaScript private field, enforced at runtime too."

### Practical

**31. How do you type API responses safely?**
> "Don't just cast `res.json()` to a type. Validate with a schema like Zod and infer the TypeScript type from that schema, so the static type and runtime check never drift apart. Or generate types and clients from an OpenAPI or GraphQL schema."

**32. Why is `catch (e)` typed as `unknown`?**
> "Because anything can be thrown — not just `Error` objects. With `useUnknownInCatchVariables`, I must narrow with `instanceof Error` or a helper before reading properties."

**33. How do you type a function whose return type depends on its input?**
> "Generics first — `function get<K extends keyof T>(key: K): T[K]`. If that's not expressible, overloads or conditional return types."

**34. What's the difference between `Object`, `object` and `{}`?**
> "`object` means any non-primitive. `{}` means any value except null and undefined — not 'empty object'. `Object` is the wrapper interface; avoid it. For 'any object with unknown keys' use `Record<string, unknown>`."

**35. What is `noUncheckedIndexedAccess`?**
> "It makes index access like `arr[i]` or `record[key]` return `T | undefined`, forcing you to handle missing elements — catching a whole class of runtime errors."

**36. How does TypeScript handle `this`?**
> "You can declare a fake first parameter `this: Type` in functions to type `this`; `noImplicitThis` errors when `this` would be `any`. Arrow functions capture lexical `this`."

**37. What is `verbatimModuleSyntax`?**
> "It makes TypeScript emit imports and exports exactly as written, dropping only those marked with `type`. It forces `import type` for type-only imports, which keeps behaviour predictable with bundlers and ESM/CommonJS interop."

**38. How do you migrate a JavaScript project to TypeScript?**
> "Add TypeScript with `allowJs` and maybe `checkJs`, set up the build and CI type-check, then convert files incrementally — starting with leaf utilities and shared types — install `@types` packages, add types at module boundaries first, use `unknown` rather than `any` where unsure, and tighten strictness flags over time."

**39. What are project references?**
> "A way to split a large codebase into smaller TypeScript projects with `composite: true` and `references`, built with `tsc -b`. It enables incremental builds and enforces boundaries — common in monorepos."

**40. What's new in recent TypeScript versions?**
> "`satisfies` in 4.9; standard decorators and `const` type parameters in 5.0; `using` declarations for resource management in 5.2; `NoInfer` in 5.4; inferred type predicates in 5.5; `erasableSyntaxOnly` for Node's type stripping in 5.8. TypeScript 6.0 turned on strict by default and deprecated legacy options like `baseUrl`, `moduleResolution: node` and ES5 targets. TypeScript 7.0 is the native Go rewrite of the compiler — around ten times faster — and removes those deprecated options."

**41. How do bundlers like Vite handle TypeScript?**
> "They transpile TypeScript with esbuild or SWC by stripping types, without type-checking, for speed. So I run `tsc --noEmit` in CI and rely on the editor for errors. `isolatedModules` ensures each file can be transpiled independently."

**42. How would you type a deep partial or deep readonly?**
```ts
type DeepPartial<T> = { [K in keyof T]?: T[K] extends object ? DeepPartial<T[K]> : T[K] };
```
> "…with care for arrays and functions in production versions."

**43. What is distributive conditional type behaviour?**
> "When a conditional type checks a naked type parameter and receives a union, it's applied to each member separately and the results are unioned — `ToArray<string | number>` becomes `string[] | number[]`. Wrapping in a tuple `[T] extends [U]` disables distribution."

**44. When would you use `unknown` over generics?**
> "When the function doesn't need to relate input and output types. If a type parameter appears only once in the signature, `unknown` is clearer."

**45. What are the limits of TypeScript's safety?**
> "Types are erased, so external data must be validated; `any`, `as` and `!` bypass checks; arrays are covariant; method parameters are bivariant; index access is unchecked without `noUncheckedIndexedAccess`; and declaration files can be wrong. TypeScript reduces bugs a lot, but it isn't a substitute for tests and runtime validation."

---

# 24. Type Challenges & Exercises

Try each before opening the solution. (More at github.com/type-challenges/type-challenges.)

**1. `Last<T>` — last element type of a tuple.**

<details><summary>Solution</summary>

```ts
type Last<T extends readonly unknown[]> = T extends [...unknown[], infer L] ? L : never;
type L1 = Last<[1, 2, 3]>;   // 3
```

</details>

**2. `TupleToUnion<T>`.**

<details><summary>Solution</summary>

```ts
type TupleToUnion<T extends readonly unknown[]> = T[number];
type U1 = TupleToUnion<["a", "b"]>;   // "a" | "b"
```

</details>

**3. `IsNever<T>`.**

<details><summary>Solution</summary>

```ts
type IsNever<T> = [T] extends [never] ? true : false;   // brackets prevent distribution over never
type N1 = IsNever<never>;    // true
type N2 = IsNever<string>;   // false
```

</details>

**4. `Mutable<T>` (remove readonly deeply).**

<details><summary>Solution</summary>

```ts
type DeepMutable<T> = { -readonly [K in keyof T]: T[K] extends object ? DeepMutable<T[K]> : T[K] };
```

</details>

**5. `PromiseAll` typing — `Promise.all`-like function preserving tuple types.**

<details><summary>Solution</summary>

```ts
declare function promiseAll<const T extends readonly unknown[]>(
  values: T,
): Promise<{ -readonly [K in keyof T]: Awaited<T[K]> }>;

async function demo() {
  const r = await promiseAll([Promise.resolve(1), "x", Promise.resolve(true)]);
  // r: [number, "x", boolean]
}
```

</details>

**6. `Trim<S>` for string literal types.**

<details><summary>Solution</summary>

```ts
type Whitespace = " " | "\n" | "\t";
type TrimLeft<S extends string> = S extends `${Whitespace}${infer R}` ? TrimLeft<R> : S;
type TrimRight<S extends string> = S extends `${infer R}${Whitespace}` ? TrimRight<R> : S;
type Trim<S extends string> = TrimLeft<TrimRight<S>>;
type T1 = Trim<"  hello  ">;   // "hello"
```

</details>

**7. `CamelCase<S>` — `"user_first_name"` → `"userFirstName"`.**

<details><summary>Solution</summary>

```ts
type CamelCase<S extends string> = S extends `${infer Head}_${infer Tail}`
  ? `${Head}${Capitalize<CamelCase<Tail>>}`
  : S;
type C1 = CamelCase<"user_first_name">;   // "userFirstName"
```

</details>

**8. `UnionToIntersection<U>` (advanced).**

<details><summary>Solution</summary>

```ts
type UnionToIntersection<U> = (U extends unknown ? (arg: U) => void : never) extends (arg: infer I) => void
  ? I
  : never;
type I1 = UnionToIntersection<{ a: 1 } | { b: 2 }>;   // { a: 1 } & { b: 2 }
```

Uses contravariance of function parameters: inferring from a union of functions' parameter positions produces an intersection.

</details>

**9. Practical: type a `groupBy` function.**

<details><summary>Solution</summary>

```ts
function groupBy<T, K extends PropertyKey>(items: readonly T[], getKey: (item: T) => K): Partial<Record<K, T[]>> {
  const result: Partial<Record<K, T[]>> = {};
  for (const item of items) {
    const key = getKey(item);
    (result[key] ??= []).push(item);
  }
  return result;
}

const byRole = groupBy([{ name: "a", role: "admin" as const }], (u) => u.role);
// Partial<Record<"admin", { name: string; role: "admin" }[]>>
```

</details>

**10. Practical: make this compile safely without `any`.**

```ts
function getValue(obj: Record<string, unknown>, key: string) {
  return obj[key].toString();   // ❌ obj[key] is unknown
}
```

<details><summary>Solution</summary>

```ts
function getValueSafe(obj: Record<string, unknown>, key: string): string {
  const value = obj[key];
  if (value === undefined || value === null) return "";
  return String(value);
}
```

</details>

---

## ✅ Final Revision Checklist

- [ ] Explain what TS is, type erasure and structural typing
- [ ] Configure a modern `tsconfig` (strict, noUncheckedIndexedAccess, bundler/nodenext, verbatimModuleSyntax)
- [ ] Use primitives, arrays, tuples, literal types, `as const`
- [ ] Explain `any` vs `unknown` vs `never` vs `void`
- [ ] Choose between `type` and `interface`
- [ ] Model state with discriminated unions + exhaustive `never` checks
- [ ] Narrow with typeof/instanceof/in/predicates/assertion functions
- [ ] Write generic functions with constraints and `keyof`
- [ ] Use and implement utility types (Partial, Pick, Omit, Record, ReturnType, Awaited)
- [ ] Use mapped, conditional, `infer`, template literal and recursive types
- [ ] Explain `satisfies`, `as`, `!` and when to use each
- [ ] Use classes with modifiers, abstract classes, `#private`
- [ ] Prefer literal unions / `as const` over enums — and explain why
- [ ] Write `.d.ts`, `declare module`, module augmentation
- [ ] Validate runtime data with Zod; use branded types
- [ ] Fix TS2322, TS2339, TS2532, TS7006, TS7053, TS18046
- [ ] Summarize TS 5.x → 7.0 changes

> **Golden habits:** strict on · `unknown` over `any` · annotate boundaries, infer the rest · discriminated unions for state · validate external data · avoid `as` and `!` · `satisfies` for config · literal unions over enums · `tsc --noEmit` in CI.
