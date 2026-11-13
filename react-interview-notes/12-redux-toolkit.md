# 12 — Redux Toolkit & RTK Query

> **Interview + Understanding Guide** — Roadmap Section 12
>
> Goal: Understand Redux's core principles and data flow, write modern Redux with Redux Toolkit (store, slices, thunks, selectors, entity adapters, middleware), and use RTK Query for server state.

---

# 1. Redux Core Concepts ⭐

### Three principles

1. **Single source of truth** — the whole app state in one store.
2. **State is read-only** — change it only by dispatching **actions** (plain objects describing what happened).
3. **Changes are made with pure reducers** — `(state, action) => newState`.

### Data flow

```text
UI event
   │ dispatch({ type: "cart/added", payload: product })
   ▼
Middleware (thunks, logging, analytics)
   ▼
Reducers compute new state (pure, immutable)
   ▼
Store notifies subscribers
   ▼
Components re-render if their selected data changed (useSelector)
```

| Term | Meaning |
| --- | --- |
| **Store** | Holds state; `getState()`, `dispatch()`, `subscribe()` |
| **Action** | `{ type: "todos/added", payload }` |
| **Action creator** | Function returning an action |
| **Reducer** | Pure function computing next state |
| **Selector** | Function extracting/deriving data from state |
| **Middleware** | Intercepts actions between dispatch and reducer (async, logging) |
| **Thunk** | A function dispatched instead of an object, for async logic |

### Why Redux Toolkit?

Classic Redux needed action type constants, action creators, switch reducers, manual immutable updates, store setup with middleware & devtools. **RTK** removes the boilerplate and is the **officially recommended** way to write Redux.

---

# 2. Store Setup

```js
// app/store.js
import { configureStore } from "@reduxjs/toolkit";
import cartReducer from "../features/cart/cartSlice";
import { api } from "../services/api";

export const store = configureStore({
  reducer: {
    cart: cartReducer,
    [api.reducerPath]: api.reducer,
  },
  middleware: (getDefault) => getDefault().concat(api.middleware),
  // devtools enabled automatically in development
});
```

`configureStore` adds: Redux DevTools, thunk middleware, and dev checks for **accidental mutations** and **non-serializable values**.

```jsx
// main.jsx
import { Provider } from "react-redux";
<Provider store={store}>
  <App />
</Provider>;
```

### TypeScript typed hooks

```ts
export type RootState = ReturnType<typeof store.getState>;
export type AppDispatch = typeof store.dispatch;

export const useAppDispatch = useDispatch.withTypes<AppDispatch>();
export const useAppSelector = useSelector.withTypes<RootState>();
```

---

# 3. `createSlice` ⭐

```js
// features/cart/cartSlice.js
import { createSlice } from "@reduxjs/toolkit";

const initialState = { items: [], coupon: null };

const cartSlice = createSlice({
  name: "cart",
  initialState,
  reducers: {
    itemAdded(state, action) {
      const existing = state.items.find((i) => i.id === action.payload.id);
      if (existing) existing.qty += 1;          // "mutating" — Immer produces an immutable update
      else state.items.push({ ...action.payload, qty: 1 });
    },
    itemRemoved(state, action) {
      state.items = state.items.filter((i) => i.id !== action.payload);
    },
    couponApplied: {
      reducer(state, action) {
        state.coupon = action.payload;
      },
      prepare(code) {                            // customize the action payload
        return { payload: { code, appliedAt: Date.now() } };
      },
    },
    cartCleared: () => initialState,             // return a new state to replace
  },
});

export const { itemAdded, itemRemoved, couponApplied, cartCleared } = cartSlice.actions;
export default cartSlice.reducer;
```

- Generates **action types** (`"cart/itemAdded"`) and **action creators** automatically.
- Uses **Immer**: write "mutating" code; you get immutable results. Either mutate the draft **or** return new state — not both.

### Using it

```jsx
import { useSelector, useDispatch } from "react-redux";

function CartButton({ product }) {
  const dispatch = useDispatch();
  const count = useSelector((state) => state.cart.items.length);
  return <button onClick={() => dispatch(itemAdded(product))}>Add ({count})</button>;
}
```

---

# 4. Selectors & Memoization ⭐

`useSelector` re-runs the selector after every dispatch and **re-renders the component if the result changed** (`===` by default).

```js
// ❌ returns a new array every time → re-renders on EVERY dispatch
const done = useSelector((s) => s.todos.filter((t) => t.done));
```

### `createSelector` (Reselect) — memoized selectors

```js
import { createSelector } from "@reduxjs/toolkit";

const selectItems = (state) => state.cart.items;
const selectCoupon = (state) => state.cart.coupon;

export const selectSubtotal = createSelector([selectItems], (items) =>
  items.reduce((sum, i) => sum + i.price * i.qty, 0),
);

export const selectTotal = createSelector([selectSubtotal, selectCoupon], (subtotal, coupon) =>
  coupon ? subtotal * 0.9 : subtotal,
);
```

Recomputes only when inputs change; returns the **same reference** otherwise.

### Other tips

- Select the **smallest** piece of data needed.
- Multiple `useSelector` calls are fine.
- `shallowEqual` as second arg for object results: `useSelector(sel, shallowEqual)`.
- Keep derived data in selectors, not in state.

---

# 5. Async Logic: `createAsyncThunk` ⭐

```js
import { createAsyncThunk, createSlice } from "@reduxjs/toolkit";

export const fetchProducts = createAsyncThunk(
  "products/fetch",
  async (category, { signal, rejectWithValue }) => {
    const res = await fetch(`/api/products?category=${category}`, { signal });
    if (!res.ok) return rejectWithValue(await res.json());
    return res.json();
  },
  {
    condition: (category, { getState }) => getState().products.status !== "loading", // dedupe
  },
);

const productsSlice = createSlice({
  name: "products",
  initialState: { items: [], status: "idle", error: null },
  reducers: {},
  extraReducers: (builder) => {
    builder
      .addCase(fetchProducts.pending, (state) => {
        state.status = "loading";
        state.error = null;
      })
      .addCase(fetchProducts.fulfilled, (state, action) => {
        state.status = "succeeded";
        state.items = action.payload;
      })
      .addCase(fetchProducts.rejected, (state, action) => {
        state.status = "failed";
        state.error = action.payload ?? action.error.message;
      });
  },
});
```

```jsx
useEffect(() => {
  const promise = dispatch(fetchProducts(category));
  return () => promise.abort(); // cancels via the signal
}, [category, dispatch]);

// Await result in a handler
const result = await dispatch(fetchProducts("books")).unwrap(); // throws on rejection
```

### Thunks (manual)

```js
export const checkout = () => async (dispatch, getState) => {
  const { items } = getState().cart;
  await api.checkout(items);
  dispatch(cartCleared());
};
```

---

# 6. `createEntityAdapter` (normalized collections)

```js
import { createEntityAdapter, createSlice } from "@reduxjs/toolkit";

const todosAdapter = createEntityAdapter({ sortComparer: (a, b) => a.createdAt - b.createdAt });

const todosSlice = createSlice({
  name: "todos",
  initialState: todosAdapter.getInitialState({ filter: "all" }), // { ids: [], entities: {}, filter }
  reducers: {
    todoAdded: todosAdapter.addOne,
    todoUpdated: todosAdapter.updateOne,   // payload: { id, changes }
    todoRemoved: todosAdapter.removeOne,
    todosReceived: todosAdapter.setAll,
  },
});

export const { selectAll: selectAllTodos, selectById: selectTodoById } =
  todosAdapter.getSelectors((state) => state.todos);
```

---

# 7. Middleware & Listener Middleware

```js
// Custom middleware
const logger = (store) => (next) => (action) => {
  console.log("dispatching", action.type);
  const result = next(action);
  console.log("next state", store.getState());
  return result;
};
```

```js
// Listener middleware: react to actions with side effects (lighter alternative to sagas)
import { createListenerMiddleware } from "@reduxjs/toolkit";

export const listener = createListenerMiddleware();
listener.startListening({
  actionCreator: itemAdded,
  effect: async (action, api) => {
    analytics.track("add_to_cart", action.payload);
    localStorage.setItem("cart", JSON.stringify(api.getState().cart));
  },
});
// configureStore({ middleware: (gDM) => gDM().prepend(listener.middleware) })
```

Older alternatives: **redux-saga** (generators, powerful for complex flows), **redux-observable** (RxJS).

---

# 8. RTK Query ⭐ (server state in Redux)

Data fetching + caching built into RTK — similar to TanStack Query.

```js
// services/api.js
import { createApi, fetchBaseQuery } from "@reduxjs/toolkit/query/react";

export const api = createApi({
  reducerPath: "api",
  baseQuery: fetchBaseQuery({
    baseUrl: "/api",
    prepareHeaders: (headers, { getState }) => {
      const token = getState().auth.token;
      if (token) headers.set("authorization", `Bearer ${token}`);
      return headers;
    },
  }),
  tagTypes: ["Post"],
  endpoints: (build) => ({
    getPosts: build.query({
      query: (page = 1) => `posts?page=${page}`,
      providesTags: (result = []) => [
        ...result.map(({ id }) => ({ type: "Post", id })),
        { type: "Post", id: "LIST" },
      ],
    }),
    getPost: build.query({
      query: (id) => `posts/${id}`,
      providesTags: (r, e, id) => [{ type: "Post", id }],
    }),
    addPost: build.mutation({
      query: (body) => ({ url: "posts", method: "POST", body }),
      invalidatesTags: [{ type: "Post", id: "LIST" }], // refetch list automatically
    }),
    updatePost: build.mutation({
      query: ({ id, ...patch }) => ({ url: `posts/${id}`, method: "PATCH", body: patch }),
      async onQueryStarted({ id, ...patch }, { dispatch, queryFulfilled }) {
        // Optimistic update
        const patchResult = dispatch(
          api.util.updateQueryData("getPost", id, (draft) => {
            Object.assign(draft, patch);
          }),
        );
        try {
          await queryFulfilled;
        } catch {
          patchResult.undo(); // rollback
        }
      },
      invalidatesTags: (r, e, { id }) => [{ type: "Post", id }],
    }),
  }),
});

export const { useGetPostsQuery, useGetPostQuery, useAddPostMutation, useUpdatePostMutation } = api;
```

```jsx
function Posts() {
  const [page, setPage] = useState(1);
  const { data = [], isLoading, isFetching, error, refetch } = useGetPostsQuery(page, {
    pollingInterval: 0,
    refetchOnFocus: true,
  });
  const [addPost, { isLoading: isAdding }] = useAddPostMutation();

  if (isLoading) return <Spinner />;
  if (error) return <ErrorView error={error} />;
  return (
    <>
      {data.map((p) => <PostRow key={p.id} post={p} />)}
      <button disabled={isAdding} onClick={() => addPost({ title: "New" })}>Add</button>
    </>
  );
}
```

Features: caching keyed by args, **deduping**, tag-based **invalidation**, polling, refetch on focus/reconnect, optimistic updates, code-generated hooks, `skip` / `skipToken` for conditional queries, lazy queries.

### RTK Query vs TanStack Query

| RTK Query | TanStack Query |
| --- | --- |
| Lives in the Redux store (devtools, middleware) | Independent cache |
| Endpoints defined centrally (`createApi`) | Queries defined where used (or in hooks) |
| Tag-based invalidation | Query-key-based invalidation |
| Great if you already use Redux | Great standalone; richer infinite/pagination APIs |

---

# 9. Best Practices (Redux Style Guide highlights)

- Use **RTK** (`configureStore`, `createSlice`); don't write classic Redux by hand.
- Organize by **feature folders** (`features/cart/cartSlice.js`).
- Keep state **minimal & serializable** (no Promises, class instances, DOM nodes, functions).
- Model actions as **events** (`orderPlaced`), not setters (`setOrderStatus`).
- Put as much logic as possible in **reducers** (pure, testable).
- **Normalize** collections.
- Use **memoized selectors** for derived data.
- **Don't put everything in Redux** — local UI state stays local; server data in RTK Query/TanStack Query; form state in RHF.

---

# 10. Interview Answers

**Explain the Redux data flow.**

> "The UI dispatches an action describing what happened. It passes through middleware, which can handle async work, then the root reducer computes the next state as a pure function. The store notifies subscribers, and components using `useSelector` re-render only if the data they select changed."

**Why Redux Toolkit over classic Redux?**

> "RTK removes the boilerplate: `createSlice` generates action creators and types, Immer lets me write simple 'mutating' reducers safely, `configureStore` sets up devtools, thunks and mutation checks, and RTK Query handles data fetching and caching. It's the officially recommended way to write Redux."

**How do you avoid unnecessary re-renders with Redux?**

> "Select the smallest piece of state, avoid selectors that return new objects or arrays every time, and use `createSelector` for derived data so it's memoized. For object results I can pass `shallowEqual`."

---

# 11. Key Takeaways

- Redux: single store, actions, pure reducers, middleware, selectors.
- RTK: `configureStore`, `createSlice` (Immer), `createAsyncThunk`, `createEntityAdapter`, listener middleware.
- `useSelector` re-renders on reference change → memoized selectors.
- RTK Query: caching, invalidation via tags, optimistic updates.
- Don't put everything in Redux.

## Readiness Checklist

- [ ] Draw Redux data flow
- [ ] Write a slice with prepare callback and extraReducers
- [ ] Write memoized selectors and explain why
- [ ] Implement an async thunk with abort
- [ ] Build CRUD with RTK Query + optimistic update
- [ ] Write a custom logger middleware
