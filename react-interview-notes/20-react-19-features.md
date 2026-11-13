# 20 — React 19 Features

> **Interview + Understanding Guide** — Roadmap Section 20
>
> Goal: Know what changed in React 19 (released Dec 2024; 19.1 and 19.2 in 2025): Actions, new hooks, `use`, Server Components & Server Actions, ref as a prop, document metadata, resource preloading, the React Compiler, and removed legacy APIs.

---

# 1. Overview

| Area | What's new |
| --- | --- |
| **Actions** | Async functions in transitions; automatic pending/error/optimistic handling; `<form action={fn}>` |
| **New hooks** | `useActionState`, `useOptimistic`, `useFormStatus` (react-dom), `use` |
| **Server** | Server Components & Server Actions stable (via frameworks) |
| **DX** | `ref` as a prop, `<Context>` as provider, ref cleanup functions, better hydration error diffs |
| **Document** | `<title>`, `<meta>`, `<link>` rendered anywhere are hoisted to `<head>`; stylesheet precedence; async scripts |
| **Resources** | `preload`, `preinit`, `prefetchDNS`, `preconnect` APIs |
| **Errors** | `onCaughtError`, `onUncaughtError` root options; errors no longer re-thrown twice |
| **19.1 / 19.2** | Owner stacks (debugging), `<Activity>`, `useEffectEvent`, `cacheSignal`, Performance tracks in DevTools, partial pre-rendering APIs |
| **Compiler** | React Compiler 1.0 (separate build plugin) — automatic memoization |

---

# 2. Actions ⭐

By convention, functions that use **async transitions** are called **Actions**. React manages:
- **Pending state** (starts at request, resets when done)
- **Optimistic updates** (`useOptimistic`)
- **Error handling** (errors go to error boundaries / you can return error state)
- **Forms**: `<form action>` resets uncontrolled fields after success

### Before (React 18)

```jsx
function UpdateName() {
  const [name, setName] = useState("");
  const [error, setError] = useState(null);
  const [isPending, setIsPending] = useState(false);

  const handleSubmit = async () => {
    setIsPending(true);
    const error = await updateName(name);
    setIsPending(false);
    if (error) return setError(error);
    redirect("/path");
  };
}
```

### After (React 19)

```jsx
function UpdateName() {
  const [error, submitAction, isPending] = useActionState(async (previousState, formData) => {
    const error = await updateName(formData.get("name"));
    if (error) return error;
    redirect("/path");
    return null;
  }, null);

  return (
    <form action={submitAction}>
      <input type="text" name="name" />
      <button type="submit" disabled={isPending}>Update</button>
      {error && <p>{error}</p>}
    </form>
  );
}
```

`<form action>`, `<button formAction>`, `<input formAction>` accept functions.

---

# 3. New Hooks (summary)

```jsx
// useActionState — state for an action
const [state, formAction, isPending] = useActionState(fn, initialState, permalink?);

// useFormStatus — pending status of the parent form (design-system buttons)
const { pending, data, method, action } = useFormStatus();

// useOptimistic — temporary optimistic state during an action
const [optimisticState, addOptimistic] = useOptimistic(state, updateFn);

// use — read a promise or context during render (can be conditional)
const value = use(promiseOrContext);
```

Detailed examples: `07-advanced-hooks.md` §11.

---

# 4. `use` ⭐

```jsx
function Message({ messagePromise }) {
  const message = use(messagePromise); // suspends until resolved
  return <p>{message.text}</p>;
}

function App() {
  const messagePromise = fetchMessage(); // ⚠️ in real apps: create in a Server Component/loader or cache it
  return (
    <ErrorBoundary fallback={<p>Error</p>}>
      <Suspense fallback={<p>Loading…</p>}>
        <Message messagePromise={messagePromise} />
      </Suspense>
    </ErrorBoundary>
  );
}
```

- Can be called inside `if`/loops (unlike other hooks) but still only in components/hooks.
- Promises created during render must be **stable** (cached) or it re-suspends forever.
- Common pattern: Server Component starts the fetch, passes the promise to a Client Component that `use`s it — streaming data without blocking the page.

---

# 5. Server Components & Server Actions

- **Server Components** render on the server (at build or request time), can be `async`, fetch data directly, and send **no JS** to the client. Client Components are marked with `"use client"`.
- **Server Actions** (`"use server"`) are server functions callable from Client Components (often via `<form action>`); the framework creates the network endpoint.

```jsx
// app/actions.js
"use server";
export async function createTodo(formData) {
  await db.todo.create({ data: { title: formData.get("title") } });
  revalidatePath("/todos");
}

// app/todos/page.jsx (Server Component)
import { createTodo } from "../actions";
export default async function Todos() {
  const todos = await db.todo.findMany();
  return (
    <>
      <ul>{todos.map((t) => <li key={t.id}>{t.title}</li>)}</ul>
      <form action={createTodo}>
        <input name="title" />
        <button>Add</button>
      </form>
    </>
  );
}
```

Used through frameworks (Next.js App Router, React Router framework mode experiments, Waku). Deep dive: `21-ssr-hydration-server-components.md`, `22-nextjs.md`.

---

# 6. `ref` as a Prop ⭐

```jsx
function MyInput({ placeholder, ref }) {
  return <input placeholder={placeholder} ref={ref} />;
}
<MyInput ref={inputRef} />
```

No more `forwardRef` for new code (it still works; a codemod migrates).

### Ref cleanup functions

```jsx
<input ref={(node) => {
  // setup
  return () => { /* cleanup on detach */ };
}} />
```

⚠️ Implicit returns from ref callbacks are now treated as cleanup: `ref={(n) => (instance = n)}` → TypeScript error; use a block body.

---

# 7. `<Context>` as a Provider

```jsx
const ThemeContext = createContext("light");

<ThemeContext value="dark">   {/* instead of <ThemeContext.Provider value="dark"> */}
  <App />
</ThemeContext>
```

---

# 8. Document Metadata

Render `<title>`, `<meta>`, `<link>` anywhere — React hoists them to `<head>` (works with streaming SSR and RSC).

```jsx
function BlogPost({ post }) {
  return (
    <article>
      <title>{post.title}</title>
      <meta name="description" content={post.excerpt} />
      <meta name="author" content={post.author} />
      <link rel="canonical" href={post.url} />
      <h1>{post.title}</h1>
    </article>
  );
}
```

Replaces `react-helmet` for simple cases (frameworks still provide richer metadata APIs).

### Stylesheets with precedence & async scripts

```jsx
<link rel="stylesheet" href="/card.css" precedence="default" /> {/* deduped, ordered, Suspense waits for it */}
<script async src="https://widget.example.com/x.js" />           {/* deduped, rendered anywhere */}
```

---

# 9. Resource Preloading APIs

```jsx
import { prefetchDNS, preconnect, preload, preinit } from "react-dom";

function App() {
  preinit("https://example.com/analytics.js", { as: "script" }); // load & execute eagerly
  preload("/fonts/inter.woff2", { as: "font", type: "font/woff2", crossOrigin: "anonymous" });
  preconnect("https://api.example.com");
  prefetchDNS("https://cdn.example.com");
  return …;
}
```

---

# 10. Hydration Improvements

- Hydration mismatch errors show a **single diff** of what differed.
- Third-party scripts/extensions inserting elements in `<head>`/`<body>` no longer cause mismatch errors as often (React skips unexpected tags).
- Better support for custom elements (Web Components): properties vs attributes handled correctly.

---

# 11. React Compiler

Build-time plugin (Babel/Vite/Next.js/Expo) that **auto-memoizes** components and hooks. v1.0 stable (Oct 2025). Requires code following the Rules of React; ships with an ESLint rule set. See `15-performance-optimization.md` §5.

---

# 12. Removed / Deprecated (React 19 breaking changes) ⭐

| Removed | Replacement |
| --- | --- |
| `ReactDOM.render`, `ReactDOM.hydrate` | `createRoot`, `hydrateRoot` |
| `unmountComponentAtNode` | `root.unmount()` |
| `ReactDOM.findDOMNode` | refs |
| `propTypes` checks & `defaultProps` for function components | TypeScript; ES default parameters |
| Legacy Context (`contextTypes`, `getChildContext`) | `createContext` |
| String refs (`ref="input"`) | callback/object refs |
| Module pattern factories, `createFactory` | JSX |
| `react-test-renderer/shallow` | React Testing Library |
| `act` from `react-dom/test-utils` | `act` from `react` |
| UMD builds | ESM CDNs (esm.sh) |
| `element.ref` access | `element.props.ref` |

Other changes:
- Errors in render are reported via `window.reportError` / console once (not re-thrown twice).
- `useRef()` requires an argument in TypeScript (`useRef(null)`).
- Suspense commits fallbacks sooner (sibling pre-warming).
- New JSX transform is required.

Upgrade path: React 18.3 adds warnings for everything removed in 19 → fix warnings → upgrade → run codemods (`npx codemod react/19/migration-recipe`).

---

# 13. React 19.2 Highlights

- **`<Activity>`** — hide/show subtrees while preserving state.
- **`useEffectEvent`** — stable; separate events from effects.
- **`cacheSignal`** (RSC) — know when a `cache()` lifetime ends to abort work.
- **Performance tracks** in Chrome DevTools (Scheduler & Components tracks).
- **Partial pre-rendering** primitives in `react-dom/server` (prerender static shell, resume later).
- Batched Suspense boundary reveals during SSR streaming.

---

# 14. Interview Answers

**What's new in React 19?**

> "The headline is Actions: async functions inside transitions with built-in pending state, errors, and optimistic updates, plus forms accepting functions as `action`. New hooks support that — `useActionState`, `useFormStatus`, `useOptimistic` — and `use` for reading promises and context during render. Server Components and Server Actions are stable for frameworks. DX improvements include `ref` as a regular prop, `<Context>` as a provider, ref cleanup, document metadata in components, and better hydration errors. Legacy APIs like `ReactDOM.render`, string refs and `propTypes` were removed. Separately, the React Compiler reached 1.0."

**How would you upgrade a React 18 app to 19?**

> "Upgrade to 18.3 first and fix all deprecation warnings, move to `createRoot` if needed, replace string refs, legacy context, `findDOMNode` and `defaultProps` on function components, check third-party libraries for compatibility, run the official codemods, update TypeScript types, and run the test suite and a production-build smoke test."

---

# 15. Key Takeaways

- Actions = async transitions with automatic pending/error/optimistic handling.
- Hooks: `useActionState`, `useFormStatus`, `useOptimistic`, `use`.
- Server Components + Server Actions via frameworks.
- `ref` as prop, `<Context>` provider, ref cleanup.
- Metadata/stylesheets/scripts in components; resource preloading APIs.
- Removed: `ReactDOM.render`, string refs, legacy context, `propTypes`/function `defaultProps`, `findDOMNode`.
- 19.2: `<Activity>`, `useEffectEvent`; Compiler 1.0.

## Readiness Checklist

- [ ] Convert a React 18 form to `useActionState` + `useFormStatus`
- [ ] Add optimistic UI with `useOptimistic`
- [ ] Use `use()` with Suspense and an error boundary
- [ ] List 6 removed APIs and replacements
- [ ] Explain the upgrade path
