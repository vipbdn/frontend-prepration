# 19 — Concurrent React

> **Interview + Understanding Guide** — Roadmap Section 19
>
> Goal: Explain what "concurrent rendering" means, how React prioritizes updates, and how to use transitions, deferred values, Suspense, and automatic batching to keep apps responsive.

---

# 1. The Problem Concurrent Rendering Solves

Before React 18, rendering was **synchronous and uninterruptible**: once React started rendering an update, it had to finish — even if the user typed in the meantime.

```text
User types "a" → React renders 5,000 filtered rows (200ms, main thread blocked)
User types "b" → keystroke waits… input feels laggy
```

**Concurrent React** (React 18+, enabled by `createRoot`) makes rendering **interruptible**:
- React can **pause** a render, handle a more urgent update, then **resume or discard** the old work.
- Multiple versions of the UI can be "in progress" at once; only the finished one is committed.

> Concurrency ≠ parallelism. It's still one thread; React just **time-slices** and **prioritizes** work (Fiber + Scheduler, see `09-rendering-reconciliation-fiber.md`).

---

# 2. Urgent vs Non-Urgent Updates ⭐

| Urgent (default) | Non-urgent (transitions) |
| --- | --- |
| Typing, clicking, pressing | Rendering search results, switching tabs, filtering a big list, navigation |
| User expects immediate feedback | A slight delay is acceptable |
| Never interrupted | Interruptible; old transitions discarded when new ones arrive |

You opt into concurrency **per update** by marking it as a transition. Everything else behaves as before.

---

# 3. `useTransition` / `startTransition` ⭐

```jsx
function FilterableList({ items }) {
  const [query, setQuery] = useState("");          // urgent: input
  const [filter, setFilter] = useState("");        // non-urgent: list
  const [isPending, startTransition] = useTransition();

  function handleChange(e) {
    setQuery(e.target.value);                      // update input immediately
    startTransition(() => {
      setFilter(e.target.value);                   // heavy re-render can be interrupted
    });
  }

  const filtered = useMemo(() => items.filter((i) => i.includes(filter)), [items, filter]);

  return (
    <>
      <input value={query} onChange={handleChange} />
      {isPending && <span>Updating…</span>}
      <List items={filtered} />
    </>
  );
}
```

Key behaviours:
- Updates in `startTransition` get **TransitionLane** priority.
- If the user types again, React **abandons** the in-progress transition render and starts over with the latest value.
- `isPending` = a transition is in progress → show subtle feedback while the **old UI stays visible**.
- With Suspense: transitions **don't show fallbacks** for already-visible content; React waits and keeps old content.
- Standalone `startTransition` (imported from `react`) when you don't need `isPending` (e.g. in a store or router).

### React 19: async transitions (Actions)

```jsx
startTransition(async () => {
  const error = await updateName(name);  // isPending stays true for the whole async function
  if (error) return setError(error);
  startTransition(() => navigate("/profile")); // state updates after await need their own startTransition
});
```

### Rules

- The function passed must be **synchronous** in React 18 (async allowed in 19).
- Don't use transitions for **controlled input values** — the input must update urgently.
- Transitions don't make rendering faster; they make the app **feel** responsive.

---

# 4. `useDeferredValue` ⭐

When you don't control the state update (e.g. value comes from props), defer the **value**:

```jsx
function SearchResults({ query }) {
  const deferredQuery = useDeferredValue(query);
  const stale = query !== deferredQuery;
  return (
    <div style={{ opacity: stale ? 0.5 : 1, transition: "opacity 0.2s" }}>
      <SlowList query={deferredQuery} />
    </div>
  );
}

const SlowList = memo(function SlowList({ query }) { /* expensive */ });
```

1. On a new `query`, React first re-renders with the **old** `deferredQuery` (cheap — `SlowList` is memoized and skipped).
2. Then it renders in the background with the new value; interrupted if `query` changes again.

React 19 adds an initial value: `useDeferredValue(value, initialValue)` — render `initialValue` first (e.g. on mount), then the real value in the background.

---

# 5. Automatic Batching

React 18 batches updates everywhere (promises, timeouts, native handlers) — fewer renders. `flushSync` opts out. See `09-rendering-reconciliation-fiber.md` §5.

---

# 6. Suspense in Concurrent React

- Suspense boundaries let React **show fallbacks for parts** of the tree while rendering the rest.
- **Streaming SSR** (`renderToPipeableStream`) sends HTML in chunks per Suspense boundary.
- **Selective hydration**: React hydrates boundaries independently and **prioritizes the part the user interacts with**.

See `18-error-boundaries-suspense-lazy.md` and `21-ssr-hydration-server-components.md`.

---

# 7. Tearing & `useSyncExternalStore`

**Tearing**: during an interruptible render, an external mutable store changes, so some components render with the old value and others with the new — inconsistent UI.

`useSyncExternalStore` prevents this by forcing a **synchronous** re-render if the store changed mid-render. All modern state libraries (Redux, Zustand) use it.

---

# 8. Other Concurrent-Era APIs

| API | Purpose |
| --- | --- |
| `<Activity mode="hidden">` (React 19.2) | Hide a subtree while **preserving its state** and deprioritizing its updates (pre-render tabs, keep back-navigation state) |
| `useOptimistic` | Optimistic state during async transitions |
| `<ViewTransition>` (experimental/canary) | Animate between UI states with the View Transitions API |

```jsx
import { Activity } from "react";

<Activity mode={tab === "inbox" ? "visible" : "hidden"}>
  <Inbox /> {/* keeps scroll/state when hidden; effects are cleaned up while hidden */}
</Activity>
```

---

# 9. Choosing the Right Tool

| Situation | Tool |
| --- | --- |
| Heavy re-render caused by your own state update | `useTransition` |
| Heavy child re-render from a value you receive | `useDeferredValue` + `memo` |
| Too many API calls while typing | Debounce |
| CPU-heavy computation (not rendering) | Web Worker |
| Long list | Virtualization |
| Don't want spinners replacing visible content on navigation | Transition (routers do this) |

---

# 10. Interview Answers

**What is concurrent rendering?**

> "It's React 18's ability to prepare multiple versions of the UI and interrupt rendering. Rendering is split into units of work, so React can pause a low-priority render, handle urgent input like typing, then resume or throw away the stale work. It's opt-in per update through transitions, and it powers Suspense features like streaming SSR and selective hydration. It's still single-threaded — it's scheduling, not parallelism."

**When would you use useTransition?**

> "When a state update triggers an expensive re-render that isn't urgent — filtering a big list, switching tabs, navigating. Wrapping it in `startTransition` keeps the input or click responsive, lets React interrupt the work if a newer update comes in, keeps the old UI visible instead of showing Suspense fallbacks, and gives me `isPending` for feedback."

---

# 11. Key Takeaways

- Concurrent = interruptible, prioritized rendering on one thread (Fiber + Scheduler).
- Urgent updates by default; mark non-urgent work with transitions.
- `useTransition` (you own the update) vs `useDeferredValue` (you receive the value; memoize the child).
- Transitions keep old UI during Suspense; React 19 supports async transitions.
- `useSyncExternalStore` prevents tearing.
- Debounce for network, workers for CPU, virtualization for long lists.

## Readiness Checklist

- [ ] Explain concurrency vs parallelism
- [ ] Demo a laggy list fixed with useTransition
- [ ] Demo useDeferredValue with a memoized child
- [ ] Explain tearing
- [ ] Explain how transitions interact with Suspense
