# 13 — State Management 🔥🔥

> **Interview + Understanding Guide** — Roadmap Section 13
>
> Goal: Choose and implement the right state approach — local signals, signal-based service stores, NgRx Store (Redux), NgRx SignalStore, and server-state tools — and explain the trade-offs.

---

# 1. Types of State ⭐

| Type | Examples | Where |
| --- | --- | --- |
| **Local UI state** | Toggle open, input text, tab index | Component `signal()` |
| **Shared/feature state** | Cart, filters, wizard data | Service store (signals) / SignalStore |
| **Global app state** | Auth user, theme, permissions | Root service / NgRx |
| **Server state (cache)** | Products, orders from the API | Resources, `shareReplay`, TanStack Query, NgRx entities |
| **URL state** | Page, filters, selected id | Router params/query params |
| **Form state** | Values, validity | Forms API |

> First rule: **keep state as local as possible**, lift it only when shared.

---

# 2. Signal-Based Service Store ⭐⭐⭐ (default choice for most apps)

```ts
import { Service, computed, inject, signal } from "@angular/core";
import { HttpClient } from "@angular/common/http";

export interface Todo { id: number; title: string; done: boolean }
type Filter = "all" | "active" | "done";

interface TodoState { todos: Todo[]; filter: Filter; loading: boolean; error: string | null }

@Service()
export class TodoStore {
  private readonly http = inject(HttpClient);
  private readonly state = signal<TodoState>({ todos: [], filter: "all", loading: false, error: null });

  // selectors
  readonly todos = computed(() => this.state().todos);
  readonly filter = computed(() => this.state().filter);
  readonly loading = computed(() => this.state().loading);
  readonly visible = computed(() => {
    const { todos, filter } = this.state();
    return filter === "all" ? todos : todos.filter((t) => (filter === "done" ? t.done : !t.done));
  });
  readonly remaining = computed(() => this.state().todos.filter((t) => !t.done).length);

  // actions / updaters
  setFilter(filter: Filter) {
    this.state.update((s) => ({ ...s, filter }));
  }

  toggle(id: number) {
    this.state.update((s) => ({ ...s, todos: s.todos.map((t) => (t.id === id ? { ...t, done: !t.done } : t)) }));
  }

  load() {
    this.state.update((s) => ({ ...s, loading: true, error: null }));
    this.http.get<Todo[]>("/api/todos").subscribe({
      next: (todos) => this.state.update((s) => ({ ...s, todos, loading: false })),
      error: () => this.state.update((s) => ({ ...s, loading: false, error: "Failed to load" })),
    });
  }

  // optimistic update with rollback
  remove(id: number) {
    const previous = this.state().todos;
    this.state.update((s) => ({ ...s, todos: s.todos.filter((t) => t.id !== id) }));
    this.http.delete(`/api/todos/${id}`).subscribe({
      error: () => this.state.update((s) => ({ ...s, todos: previous, error: "Delete failed" })),
    });
  }
}
```

Principles: **private writable state, public read-only selectors (computed), methods as the only way to change state, immutable updates.** Provide at root for global or in component `providers` for scoped instances.

Pre-signals equivalent: `BehaviorSubject` + `select` via `map` + `distinctUntilChanged` ("observable data service").

---

# 3. NgRx Store (Redux pattern) ⭐⭐

```text
Component ──dispatch(action)──▶ Store ──▶ Reducer (pure) ──▶ new State
    ▲                              │
    └──── select(selector) ◀───────┘
               Effects listen to actions → call APIs → dispatch success/failure actions
```

```ts
// actions
export const ProductsPageActions = createActionGroup({
  source: "Products Page",
  events: { Opened: emptyProps(), "Search Changed": props<{ query: string }>() },
});
export const ProductsApiActions = createActionGroup({
  source: "Products API",
  events: { "Load Success": props<{ products: Product[] }>(), "Load Failure": props<{ error: string }>() },
});

// reducer
export interface ProductsState { products: Product[]; query: string; loading: boolean; error: string | null }
const initialState: ProductsState = { products: [], query: "", loading: false, error: null };

export const productsFeature = createFeature({
  name: "products",
  reducer: createReducer(
    initialState,
    on(ProductsPageActions.opened, (s) => ({ ...s, loading: true })),
    on(ProductsPageActions.searchChanged, (s, { query }) => ({ ...s, query })),
    on(ProductsApiActions.loadSuccess, (s, { products }) => ({ ...s, products, loading: false })),
    on(ProductsApiActions.loadFailure, (s, { error }) => ({ ...s, error, loading: false })),
  ),
  extraSelectors: ({ selectProducts, selectQuery }) => ({
    selectFiltered: createSelector(selectProducts, selectQuery, (products, q) =>
      products.filter((p) => p.name.toLowerCase().includes(q.toLowerCase())),
    ),
  }),
});

// effect (functional)
export const loadProducts = createEffect(
  (actions$ = inject(Actions), api = inject(ProductApi)) =>
    actions$.pipe(
      ofType(ProductsPageActions.opened),
      exhaustMap(() =>
        api.list({ page: 1 }).pipe(
          map((products) => ProductsApiActions.loadSuccess({ products })),
          catchError((e) => of(ProductsApiActions.loadFailure({ error: String(e) }))),
        ),
      ),
    ),
  { functional: true },
);

// setup
provideStore(), provideState(productsFeature), provideEffects({ loadProducts }), provideStoreDevtools();

// component
export class ProductsPage {
  private readonly store = inject(Store);
  protected readonly products = this.store.selectSignal(productsFeature.selectFiltered);
  constructor() { this.store.dispatch(ProductsPageActions.opened()); }
}
```

Key ideas: **single source of truth**, **state is read-only** (only actions change it), **pure reducers**, **memoized selectors**, effects isolate side effects; Redux DevTools time-travel. Extras: `@ngrx/entity` (normalized collections), `@ngrx/router-store`.

---

# 4. NgRx SignalStore ⭐⭐ (modern, lightweight)

```ts
import { signalStore, withState, withComputed, withMethods, patchState, withHooks } from "@ngrx/signals";
import { rxMethod } from "@ngrx/signals/rxjs-interop";

export const BooksStore = signalStore(
  { providedIn: "root" },
  withState({ books: [] as Book[], query: "", loading: false }),
  withComputed(({ books, query }) => ({
    filtered: computed(() => books().filter((b) => b.title.toLowerCase().includes(query().toLowerCase()))),
    count: computed(() => books().length),
  })),
  withMethods((store, api = inject(BookApi)) => ({
    setQuery(query: string) {
      patchState(store, { query });
    },
    load: rxMethod<void>(
      pipe(
        tap(() => patchState(store, { loading: true })),
        switchMap(() => api.getAll().pipe(tap((books) => patchState(store, { books, loading: false })))),
      ),
    ),
  })),
  withHooks({ onInit: (store) => store.load() }),
);

// component
protected readonly store = inject(BooksStore);
// template: {{ store.count() }} @for (b of store.filtered(); track b.id) …
```

- Signals-first, little boilerplate, extensible via `signalStoreFeature`, `withEntities`, `withProps`, events plugin.
- Can be global (`providedIn: 'root'`) or component-scoped (`providers: [BooksStore]`).

---

# 5. Server State

Data from the server is a **cache**, not app state. Needs: loading/error, dedupe, refetch, invalidation, staleness.

- `httpResource` / `resource` for component-level reads
- `shareReplay(1)` caching in services
- **TanStack Query (Angular)** for caching, background refetch, mutations, invalidation
- NgRx entity + effects in large Redux apps

---

# 6. Choosing ⭐⭐⭐

| Situation | Choice |
| --- | --- |
| Small/medium app, few shared slices | **Signals in services** |
| Need structure + less boilerplate, signal-first team | **NgRx SignalStore** |
| Large team, complex flows, strict event log, time-travel debugging, many async side effects | **NgRx Store** |
| Mostly server data | Resources / TanStack Query |
| Shareable UI state | URL |

Trade-offs of NgRx Store: boilerplate, indirection, learning curve vs predictability, tooling, scalable conventions.

---

# 7. Interview Answers

**How do you manage state in Angular?**

> "By type and scope. Local UI state lives in component signals. Shared state goes into a service store — a private writable signal, computed selectors and methods that update immutably — provided at root or at a component for scoped instances. Server data I treat as a cache with resources or a query library. For large apps with complex async flows and many developers I'd use NgRx — either the classic Store with actions, reducers, selectors and effects, or SignalStore for a lighter, signal-native approach. URL-worthy state goes in route params."

**Explain the NgRx data flow.**

> "Components dispatch actions describing events. Reducers — pure functions — take the current state and an action and return new state. Components read state through memoized selectors, now usually as signals with `selectSignal`. Side effects like API calls live in effects, which listen for actions, call services and dispatch success or failure actions. It gives a single source of truth, predictable updates and DevTools time travel."

**When would you NOT use NgRx?**

> "When the app is small or state is mostly local or server cache. The boilerplate and indirection cost more than they give; signal services or SignalStore cover it. I reach for NgRx Store when many features share complex state, there's lots of async orchestration, or the team benefits from strict conventions and tooling."

**What is NgRx SignalStore?**

> "A signal-based store from NgRx built by composing features — `withState`, `withComputed`, `withMethods`, `withHooks`, entities — with `patchState` for immutable updates and `rxMethod` for RxJS-based side effects. It's much less boilerplate than the Redux store and can be global or scoped to a component."

---

# 8. Key Takeaways

- Keep state local; lift only when shared; URL for shareable state.
- Signal service store: private state, computed selectors, methods, immutable updates.
- NgRx Store: actions → reducers → selectors, effects for side effects.
- SignalStore: composable, signal-native, low boilerplate.
- Server state is a cache.

## Readiness Checklist

- [ ] Build a todo store with signals incl. optimistic delete
- [ ] Explain the NgRx flow and write a feature with effect
- [ ] Rewrite with SignalStore
- [ ] Justify a state choice for a given app
