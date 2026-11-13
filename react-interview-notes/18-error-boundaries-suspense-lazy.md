# 18 — Error Boundaries, Suspense & Lazy

> **Interview + Understanding Guide** — Roadmap Section 18
>
> Goal: Handle errors gracefully with error boundaries, understand Suspense (code, data, streaming), lazy loading, and how to design loading/error UX boundaries.

---

# 1. Error Boundaries ⭐

Without one, a rendering error **unmounts the entire React tree** (blank screen). An **error boundary** catches errors in its child tree and renders a fallback.

```jsx
class ErrorBoundary extends React.Component {
  state = { error: null };

  static getDerivedStateFromError(error) {
    return { error }; // render fallback on next render
  }

  componentDidCatch(error, info) {
    logErrorToService(error, info.componentStack); // Sentry, Datadog…
  }

  reset = () => this.setState({ error: null });

  render() {
    if (this.state.error) {
      return this.props.fallback
        ? this.props.fallback({ error: this.state.error, reset: this.reset })
        : <p>Something went wrong.</p>;
    }
    return this.props.children;
  }
}
```

- Must be a **class component** (no hook equivalent yet). In practice use **`react-error-boundary`**.
- `getDerivedStateFromError` → update state for fallback (render phase, pure).
- `componentDidCatch` → side effects like logging (commit phase).

### What error boundaries catch ⭐

✅ Errors thrown during **rendering**, in **lifecycle methods**, constructors, and hooks bodies of the child tree. Also rejected promises read with `use()` / Suspense data libraries.

❌ Do **not** catch:
- **Event handlers** (use `try/catch` inside the handler)
- **Async code** (`setTimeout`, promise callbacks, `fetch().then`) outside rendering
- Server-side rendering errors
- Errors thrown in the boundary itself

```jsx
// Bridge an async error into the nearest boundary
function useThrowAsyncError() {
  const [, setState] = useState();
  return useCallback((error) => setState(() => { throw error; }), []);
}

const throwError = useThrowAsyncError();
useEffect(() => {
  fetchData().catch(throwError);
}, [throwError]);
```

(`react-error-boundary` provides `useErrorBoundary().showBoundary(error)` for this.)

### `react-error-boundary`

```jsx
import { ErrorBoundary } from "react-error-boundary";

function Fallback({ error, resetErrorBoundary }) {
  return (
    <div role="alert">
      <p>Failed to load: {error.message}</p>
      <button onClick={resetErrorBoundary}>Try again</button>
    </div>
  );
}

<ErrorBoundary
  FallbackComponent={Fallback}
  onError={(error, info) => logError(error, info)}
  onReset={() => queryClient.resetQueries()}
  resetKeys={[userId]}             // auto-reset when these change
>
  <UserDashboard userId={userId} />
</ErrorBoundary>
```

### Placement strategy

```text
App
├── ErrorBoundary (top-level: "Something went wrong, reload")
│   ├── Header
│   ├── ErrorBoundary (route level)
│   │   └── Page
│   │       ├── ErrorBoundary (widget level) → Chart
│   │       └── ErrorBoundary (widget level) → Comments
```

Granular boundaries keep the rest of the app usable when one widget fails. Routers provide route-level boundaries (`errorElement`, Next.js `error.js`).

### React 19 error handling options

```jsx
createRoot(container, {
  onUncaughtError: (error, info) => report(error, info),   // not caught by any boundary
  onCaughtError: (error, info) => report(error, info),     // caught by a boundary
  onRecoverableError: (error) => report(error),            // e.g. hydration mismatches React recovered from
});
```

---

# 2. Suspense ⭐

`<Suspense>` shows a **fallback** while something inside it is **not ready** ("suspends").

```jsx
<Suspense fallback={<Spinner />}>
  <Comments />
</Suspense>
```

### What can suspend?

| Source | How |
| --- | --- |
| Lazy components | `React.lazy(() => import("./X"))` |
| Data (React 19) | `use(promise)` |
| Suspense-enabled libraries | TanStack Query `useSuspenseQuery`, Relay, SWR (suspense mode), Apollo |
| Server Components | async components in RSC frameworks (Next.js) |
| Streaming SSR | server sends fallback, then streams content |

> ❌ Data fetched in `useEffect` does **not** trigger Suspense.

### How it works (conceptually)

A component that isn't ready **throws a promise** (or React tracks a pending thenable via `use`). React catches it at the nearest `<Suspense>`, renders the fallback, and retries rendering when the promise resolves. If it rejects, the nearest error boundary catches it.

### Nested boundaries → progressive reveal

```jsx
<Suspense fallback={<PageSkeleton />}>
  <ProfileHeader />
  <Suspense fallback={<PostsSkeleton />}>
    <Posts />
    <Suspense fallback={<CommentsSkeleton />}>
      <Comments />
    </Suspense>
  </Suspense>
</Suspense>
```

Design loading states around **what the user should see together**. Too many spinners = "popcorn UI"; too few = waiting on the slowest piece.

### Suspense + transitions: avoid replacing content with fallbacks

```jsx
const [isPending, startTransition] = useTransition();
function selectTab(tab) {
  startTransition(() => setTab(tab)); // keep showing the OLD tab while the new one suspends
}
```

Without a transition, switching to a suspending tab immediately shows the fallback (jarring). With it, React keeps the previous UI and `isPending` lets you dim it. Router navigations in modern routers are transitions by default.

### Reset with `key`

```jsx
<Suspense key={userId} fallback={<Spinner />}>  {/* new user → show fallback again instead of stale content */}
  <Profile userId={userId} />
</Suspense>
```

### Error + loading together

```jsx
<ErrorBoundary FallbackComponent={ErrorView}>
  <Suspense fallback={<Skeleton />}>
    <Product id={id} />
  </Suspense>
</ErrorBoundary>
```

### React 19 change: sibling pre-warming

When a component suspends, React 19 commits the fallback **immediately** without rendering remaining siblings first, then "pre-warms" (renders) the siblings in the background — faster fallbacks.

---

# 3. Lazy Loading ⭐

```jsx
import { lazy, Suspense } from "react";

const Settings = lazy(() => import("./Settings")); // must resolve to { default: Component }
const Chart = lazy(() => import("./charts").then((m) => ({ default: m.LineChart }))); // named export

function App() {
  return (
    <Suspense fallback={<PageSpinner />}>
      <Settings />
    </Suspense>
  );
}
```

Rules & tips:
- Declare `lazy()` at **module top level**, never inside a component (would re-create and reset state each render).
- Wrap in `<Suspense>` (anywhere above).
- Add an error boundary — chunk loading can fail (network, new deployment removed old chunks). A common fix: catch `ChunkLoadError` and reload the page once.
- **Preload** to remove the delay:

```jsx
const loadEditor = () => import("./Editor");
const Editor = lazy(loadEditor);

<button onMouseEnter={loadEditor} onClick={() => setShowEditor(true)}>Edit</button>
```

---

# 4. Loading UX Patterns

| Pattern | When |
| --- | --- |
| **Skeletons** | Content layout known; reduces perceived wait & CLS |
| **Spinners** | Short, unknown-layout waits; small areas |
| **Keep previous content + pending indicator** | Navigations, tab switches, filters (transitions, `keepPreviousData`) |
| **Optimistic UI** | Mutations likely to succeed |
| **Delay showing spinners** (~200–300 ms) | Avoid flashing for fast responses |
| **Progressive / streaming** | Show shell + fast parts first |

---

# 5. Interview Answers

**What are error boundaries and what don't they catch?**

> "Class components implementing `getDerivedStateFromError` and/or `componentDidCatch` that catch errors during rendering, lifecycle methods and hooks in their subtree and render a fallback instead of unmounting the whole app. They don't catch errors in event handlers, async callbacks like timeouts or promise handlers outside render, SSR, or errors in the boundary itself. I place them at app, route and widget level and log to Sentry in `componentDidCatch`."

**How does Suspense work?**

> "A component that isn't ready — a lazy component still loading, or data read with `use()` or a Suspense-enabled library — suspends. React shows the nearest `<Suspense>` fallback and retries when the underlying promise resolves; if it rejects, the nearest error boundary handles it. Wrapping updates in a transition keeps the previous UI visible instead of flashing fallbacks."

---

# 6. Key Takeaways

- Error boundaries: class components (or `react-error-boundary`), render-phase errors only.
- Handle event/async errors yourself; bridge them to boundaries if needed.
- Place boundaries at app, route and widget levels; log errors.
- Suspense: lazy components, `use(promise)`, Suspense-enabled libs, RSC/streaming — not `useEffect` fetching.
- Nested boundaries for progressive loading; transitions avoid fallback flashes; `key` to reset.
- `lazy()` at module scope + Suspense + error boundary + preloading.

## Readiness Checklist

- [ ] Write an error boundary class with reset
- [ ] List what boundaries don't catch and how to handle each
- [ ] Combine ErrorBoundary + Suspense + lazy for a route
- [ ] Explain Suspense + transitions behaviour
- [ ] Design loading UX for a dashboard with 4 widgets
