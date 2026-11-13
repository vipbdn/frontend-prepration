# 13 — Data Fetching & TanStack Query 🔥

> **Interview + Understanding Guide** — Roadmap Section 13
>
> Goal: Know every way to fetch data in React, why "fetch in useEffect" doesn't scale, and master TanStack Query (React Query): query keys, caching, staleTime vs gcTime, invalidation, mutations, optimistic updates, pagination, infinite queries, prefetching, Suspense and SSR.

---

# 1. Data Fetching Options

| Approach | When |
| --- | --- |
| `useEffect` + `fetch` | Learning, tiny apps; needs manual race/loading/error handling |
| **TanStack Query / SWR** | Most client-rendered apps |
| **RTK Query** | Apps already using Redux |
| **Router loaders** (React Router data APIs) | Fetch per route before rendering; avoids waterfalls |
| **Server Components / Next.js** | Fetch on the server; ship less JS |
| **Apollo / urql / Relay** | GraphQL |

### Problems with fetch-in-effect

```text
❌ No caching — refetch on every mount
❌ No deduplication — 3 components = 3 identical requests
❌ Race conditions & unmount handling by hand
❌ No background refresh / stale data handling
❌ No retries, pagination helpers, optimistic updates
❌ Waterfalls (parent fetch → render child → child fetch)
❌ Boilerplate loading/error state everywhere
```

---

# 2. TanStack Query Setup

```jsx
import { QueryClient, QueryClientProvider } from "@tanstack/react-query";
import { ReactQueryDevtools } from "@tanstack/react-query-devtools";

const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      staleTime: 60_000,          // data fresh for 1 minute
      retry: 2,
      refetchOnWindowFocus: true,
    },
  },
});

function Root() {
  return (
    <QueryClientProvider client={queryClient}>
      <App />
      <ReactQueryDevtools initialIsOpen={false} />
    </QueryClientProvider>
  );
}
```

---

# 3. `useQuery` ⭐⭐⭐

```jsx
import { useQuery } from "@tanstack/react-query";

async function fetchUser(id) {
  const res = await fetch(`/api/users/${id}`);
  if (!res.ok) throw new Error(`HTTP ${res.status}`); // must THROW for error state
  return res.json();
}

function UserProfile({ userId }) {
  const { data, error, isPending, isError, isFetching, refetch } = useQuery({
    queryKey: ["user", userId],          // cache key — include every variable the fn uses
    queryFn: () => fetchUser(userId),
    enabled: !!userId,                   // dependent/conditional query
  });

  if (isPending) return <Spinner />;
  if (isError) return <ErrorView message={error.message} onRetry={refetch} />;
  return (
    <>
      <h1>{data.name}</h1>
      {isFetching && <small>Refreshing…</small>}
    </>
  );
}
```

### Query keys ⭐

- An **array** that uniquely identifies the data: `["todos"]`, `["todos", { status, page }]`, `["user", id, "posts"]`.
- **Every variable used in `queryFn` must be in the key** — when it changes, a new query/cache entry is used automatically (no dependency arrays!).
- Hierarchical → enables partial invalidation (`["todos"]` invalidates all todo queries).
- Use a **query key factory** for consistency:

```js
export const todoKeys = {
  all: ["todos"],
  lists: () => [...todoKeys.all, "list"],
  list: (filters) => [...todoKeys.lists(), filters],
  detail: (id) => [...todoKeys.all, "detail", id],
};
```

`queryFn` receives `{ queryKey, signal }` — pass `signal` to `fetch` for automatic cancellation.

### Status flags

| Flag | Meaning |
| --- | --- |
| `isPending` | No data yet (first load) |
| `isFetching` | Any fetch in flight (including background refetch) |
| `isLoading` | `isPending && isFetching` |
| `isError` / `error` | Last fetch failed (after retries) |
| `isSuccess` / `data` | Have data |
| `isPlaceholderData` | Showing placeholder (e.g. previous page) |

---

# 4. Caching: `staleTime` vs `gcTime` ⭐⭐⭐

```text
fetch → data FRESH ──(staleTime)──▶ data STALE ──(no observers for gcTime)──▶ garbage collected
```

| Option | Default | Meaning |
| --- | --- | --- |
| `staleTime` | `0` | How long data is considered **fresh**. Fresh data is served from cache **without refetching**. |
| `gcTime` (was `cacheTime` in v4) | 5 minutes | How long **inactive** (no component using it) data stays in memory before being removed. |

### Stale-while-revalidate

With stale data, a new mount / window focus / reconnect shows cached data **instantly** and **refetches in the background**. Fresh data → no refetch.

### Automatic refetch triggers (for stale queries)

- New component mounts using the query (`refetchOnMount`)
- Window regains focus (`refetchOnWindowFocus`)
- Network reconnects (`refetchOnReconnect`)
- `refetchInterval` (polling)

### Choosing staleTime

- Rarely changing data (config, countries): `Infinity` or minutes
- User-editable lists: 30s–5min + invalidation after mutations
- Real-time-ish dashboards: `0` + polling or websockets

### Deduplication

Multiple components using `["user", 1]` at the same time → **one** request; all share the result.

### Retries

Failed queries retry 3 times with exponential backoff by default (`retry`, `retryDelay`). Turn off for 4xx: `retry: (count, err) => err.status >= 500 && count < 3`.

---

# 5. Mutations ⭐⭐⭐

```jsx
import { useMutation, useQueryClient } from "@tanstack/react-query";

function AddTodo() {
  const queryClient = useQueryClient();

  const mutation = useMutation({
    mutationFn: (newTodo) =>
      fetch("/api/todos", { method: "POST", body: JSON.stringify(newTodo) }).then((r) => r.json()),
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: todoKeys.lists() }); // mark stale + refetch active
    },
    onError: (err) => toast.error(err.message),
  });

  return (
    <button disabled={mutation.isPending} onClick={() => mutation.mutate({ title: "New" })}>
      {mutation.isPending ? "Adding…" : "Add"}
    </button>
  );
}
```

- `mutate(vars, { onSuccess })` (fire and forget) vs `mutateAsync(vars)` (returns a promise).
- Lifecycle: `onMutate` → `mutationFn` → `onSuccess`/`onError` → `onSettled`.

### Updating the cache after a mutation

```js
// 1. Invalidate (simplest, always correct) → refetch
queryClient.invalidateQueries({ queryKey: ["todos"] });

// 2. Write the response directly (saves a request)
queryClient.setQueryData(todoKeys.detail(updated.id), updated);
```

---

# 6. Optimistic Updates ⭐⭐⭐

Update the UI **before** the server responds; roll back on failure.

```jsx
const toggleTodo = useMutation({
  mutationFn: (todo) => api.updateTodo(todo.id, { done: !todo.done }),

  onMutate: async (todo) => {
    await queryClient.cancelQueries({ queryKey: todoKeys.lists() }); // avoid overwriting optimistic data
    const previous = queryClient.getQueryData(todoKeys.list(filters));  // snapshot

    queryClient.setQueryData(todoKeys.list(filters), (old = []) =>
      old.map((t) => (t.id === todo.id ? { ...t, done: !t.done } : t)),
    );

    return { previous }; // context for rollback
  },

  onError: (err, todo, context) => {
    queryClient.setQueryData(todoKeys.list(filters), context.previous); // rollback
    toast.error("Could not update");
  },

  onSettled: () => {
    queryClient.invalidateQueries({ queryKey: todoKeys.lists() }); // sync with server truth
  },
});
```

**Simpler alternative (v5):** render optimistic UI from `mutation.variables` while `isPending` — no cache manipulation:

```jsx
{mutation.isPending && <li style={{ opacity: 0.5 }}>{mutation.variables.title}</li>}
```

React 19's `useOptimistic` is another option for action-based flows.

---

# 7. Dependent & Parallel Queries

```jsx
// Dependent: second waits for the first
const { data: user } = useQuery({ queryKey: ["user", email], queryFn: () => getUser(email) });
const { data: projects } = useQuery({
  queryKey: ["projects", user?.id],
  queryFn: () => getProjects(user.id),
  enabled: !!user?.id,
});

// Parallel (dynamic number)
import { useQueries } from "@tanstack/react-query";
const results = useQueries({
  queries: userIds.map((id) => ({ queryKey: ["user", id], queryFn: () => getUser(id) })),
});
```

---

# 8. Pagination ⭐

```jsx
import { keepPreviousData } from "@tanstack/react-query";

function Projects() {
  const [page, setPage] = useState(1);
  const { data, isPending, isPlaceholderData } = useQuery({
    queryKey: ["projects", page],
    queryFn: () => fetchProjects(page),
    placeholderData: keepPreviousData, // keep showing page N while page N+1 loads (no flash)
  });

  return (
    <>
      {data?.items.map((p) => <Row key={p.id} project={p} />)}
      <button onClick={() => setPage((p) => Math.max(p - 1, 1))} disabled={page === 1}>Prev</button>
      <button
        onClick={() => setPage((p) => p + 1)}
        disabled={isPlaceholderData || !data?.hasMore}
      >
        Next
      </button>
    </>
  );
}
```

Prefetch the next page for instant navigation:

```jsx
useEffect(() => {
  if (data?.hasMore) {
    queryClient.prefetchQuery({ queryKey: ["projects", page + 1], queryFn: () => fetchProjects(page + 1) });
  }
}, [data, page, queryClient]);
```

---

# 9. Infinite Queries ⭐

```jsx
import { useInfiniteQuery } from "@tanstack/react-query";

function Feed() {
  const { data, fetchNextPage, hasNextPage, isFetchingNextPage, status } = useInfiniteQuery({
    queryKey: ["feed"],
    queryFn: ({ pageParam, signal }) => fetch(`/api/feed?cursor=${pageParam}`, { signal }).then((r) => r.json()),
    initialPageParam: 0,
    getNextPageParam: (lastPage) => lastPage.nextCursor ?? undefined, // undefined → no more pages
  });

  const posts = data?.pages.flatMap((p) => p.items) ?? [];

  return (
    <>
      {posts.map((p) => <Post key={p.id} post={p} />)}
      <button onClick={() => fetchNextPage()} disabled={!hasNextPage || isFetchingNextPage}>
        {isFetchingNextPage ? "Loading…" : hasNextPage ? "Load more" : "No more"}
      </button>
    </>
  );
}
```

Combine with an IntersectionObserver sentinel for infinite scroll (see `08-custom-hooks.md` §13) and virtualization for very long lists.

---

# 10. Prefetching & Initial Data

```jsx
// On hover → data ready on click
<Link to={`/users/${id}`} onMouseEnter={() =>
  queryClient.prefetchQuery({ queryKey: ["user", id], queryFn: () => fetchUser(id), staleTime: 60_000 })
}>

// Seed a detail query from list data
useQuery({
  queryKey: ["todo", id],
  queryFn: () => fetchTodo(id),
  initialData: () => queryClient.getQueryData(["todos"])?.find((t) => t.id === id),
});
```

`initialData` is treated as real cached data; `placeholderData` is shown but not cached.

---

# 11. Suspense Mode

```jsx
import { useSuspenseQuery } from "@tanstack/react-query";

function User({ id }) {
  const { data } = useSuspenseQuery({ queryKey: ["user", id], queryFn: () => fetchUser(id) });
  return <h1>{data.name}</h1>; // data is always defined
}

<ErrorBoundary fallback={<ErrorView />}>
  <Suspense fallback={<Spinner />}>
    <User id={1} />
  </Suspense>
</ErrorBoundary>
```

Watch for **waterfalls**: sibling suspense queries in the same component run sequentially — use `useSuspenseQueries` or prefetch.

---

# 12. SSR / Next.js (hydration)

```jsx
// Server: prefetch and dehydrate
const queryClient = new QueryClient();
await queryClient.prefetchQuery({ queryKey: ["posts"], queryFn: getPosts });

return (
  <HydrationBoundary state={dehydrate(queryClient)}>
    <Posts /> {/* client useQuery(["posts"]) gets data instantly, no loading state */}
  </HydrationBoundary>
);
```

Create the `QueryClient` **per request** on the server (never share between users).

---

# 13. Other Useful APIs

| API | Use |
| --- | --- |
| `select` | Transform/subscribe to part of data: `select: (d) => d.items.length` (re-render only when result changes) |
| `refetchInterval` | Polling |
| `queryClient.setQueryData` | Write to cache |
| `queryClient.removeQueries` | Remove (e.g. on logout: `queryClient.clear()`) |
| `useIsFetching` / `useIsMutating` | Global loading indicators |
| `networkMode: "offlineFirst"` | Offline support |
| Persisters | Persist cache to localStorage/IndexedDB |

---

# 14. SWR (brief)

```jsx
import useSWR from "swr";
const fetcher = (url) => fetch(url).then((r) => r.json());
const { data, error, isLoading, mutate } = useSWR(`/api/user/${id}`, fetcher);
```

Smaller API by Vercel; same stale-while-revalidate idea. TanStack Query has richer mutations, devtools, infinite/pagination helpers.

---

# 15. Interview Answers

**Why TanStack Query instead of useEffect?**

> "Server state is a cache of data owned by the server, with problems like caching, deduplication, background refresh, retries, race conditions, pagination and invalidation after mutations. Fetching in `useEffect` makes me handle all of that by hand in every component. TanStack Query handles it declaratively with query keys, so components just declare what data they need."

**staleTime vs gcTime?**

> "`staleTime` is how long data is considered fresh — while fresh, it's served from the cache without refetching. Default is zero, so cached data is shown instantly but refetched in the background on mount or focus. `gcTime` is how long unused data stays in memory before being garbage collected, five minutes by default."

**How do you do optimistic updates?**

> "In `onMutate`, I cancel outgoing queries for that key, snapshot the current cache, write the optimistic value with `setQueryData`, and return the snapshot. In `onError` I restore the snapshot, and in `onSettled` I invalidate so the cache syncs with the server."

---

# 16. Key Takeaways

- Server state needs a cache → TanStack Query / RTK Query / SWR / framework loaders.
- Query keys: arrays containing every input; hierarchical; key factories.
- `staleTime` (freshness) vs `gcTime` (memory retention); stale-while-revalidate.
- Mutations + `invalidateQueries` or `setQueryData`.
- Optimistic updates: cancel → snapshot → set → rollback → invalidate.
- `keepPreviousData` pagination, `useInfiniteQuery`, prefetching, Suspense, SSR hydration.

## Readiness Checklist

- [ ] List 6 problems with fetch-in-effect
- [ ] Explain staleTime vs gcTime with a timeline
- [ ] Build a paginated list with prefetching
- [ ] Build infinite scroll with useInfiniteQuery + IntersectionObserver
- [ ] Implement optimistic toggle with rollback
- [ ] Design a query key factory
