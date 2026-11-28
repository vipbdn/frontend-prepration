# 04 — Signals 🔥🔥🔥

> **Interview + Understanding Guide** — Roadmap Section 4
>
> Goal: Master Angular's reactive primitive — `signal`, `computed`, `effect`, `linkedSignal`, `untracked` — plus signal inputs/outputs/queries, RxJS interop, and when to use signals vs observables.

---

# 1. What is a Signal? ⭐

A **signal** is a **wrapper around a value that notifies interested consumers when the value changes**. Reading a signal (calling it like a function) **tracks** the dependency; setting it **notifies** dependents.

```ts
import { signal } from "@angular/core";

const count = signal(0);   // WritableSignal<number>

count();                   // read → 0
count.set(5);              // replace
count.update((c) => c + 1); // based on previous → 6
```

### Why Angular introduced signals (v16, stable v17–v20)

| Before (zone.js + dirty checking) | With signals |
| --- | --- |
| Angular didn't know *what* changed, so after any async event it checked the whole component tree (or OnPush subtrees) | Angular knows *exactly* which views read which signals → **fine-grained** updates |
| Relied on zone.js monkey-patching async APIs | Enables **zoneless** apps (smaller, faster, simpler stack traces) |
| RxJS for all reactivity (steep learning curve, subscriptions) | Simple synchronous reactive state; RxJS kept for async streams |
| Derived state recomputed manually / in `ngOnChanges` | `computed` derives state automatically and lazily |

> **Interview definition:** "Signals are reactive values that track where they're read. When a signal changes, Angular knows precisely which computed values, effects and templates depend on it and updates only those — enabling fine-grained change detection and zoneless apps."

---

# 2. Writable Signals

```ts
interface Todo { id: number; title: string; done: boolean }

const todos = signal<Todo[]>([]);

// ✅ immutable updates (new reference → dependents notified)
todos.update((list) => [...list, { id: 1, title: "Learn signals", done: false }]);
todos.update((list) => list.map((t) => (t.id === 1 ? { ...t, done: true } : t)));

// ❌ mutating in place doesn't notify (same reference)
todos().push({ id: 2, title: "x", done: false });
```

### Equality

Signals use `Object.is` by default; setting an equal value doesn't notify. Custom equality:

```ts
const user = signal({ id: 1, name: "Asha" }, { equal: (a, b) => a.id === b.id });
```

### Read-only exposure

```ts
@Service()
export class CartStore {
  private readonly _items = signal<CartItem[]>([]);
  readonly items = this._items.asReadonly();   // consumers can read, not write
}
```

---

# 3. `computed` — derived state ⭐

```ts
import { computed, signal } from "@angular/core";

const price = signal(100);
const quantity = signal(2);
const total = computed(() => price() * quantity());   // Signal<number> (read-only)

total();         // 200
quantity.set(3);
total();         // 300
```

- **Lazy**: computed only when read.
- **Memoized**: cached until a dependency changes.
- **Dynamic dependencies**: only signals actually read in the last run are tracked.

```ts
const showDetails = signal(false);
const details = computed(() => (showDetails() ? expensiveDetails() : null));
// expensiveDetails is a dependency only while showDetails() is true
```

⚠️ Computeds must be **pure** — no side effects, no writing signals inside them.

---

# 4. `effect` — side effects ⭐

Runs a function whenever the signals it reads change. Runs at least once.

```ts
import { Component, effect, signal } from "@angular/core";

@Component({ selector: "app-theme", template: `…` })
export class Theme {
  readonly mode = signal<"light" | "dark">("light");

  constructor() {
    effect(() => {
      document.documentElement.dataset["theme"] = this.mode();   // sync to DOM
      localStorage.setItem("theme", this.mode());               // sync to storage
    });
  }
}
```

### Rules & good practice ⭐

- Must be created in an **injection context** (constructor, field initializer) or pass `{ injector }`.
- Automatically destroyed with the component/service (or call `ref.destroy()`).
- Effects run **asynchronously**, during change detection (component effects before the view is checked).
- **Use effects for syncing with the outside world**: logging, analytics, `localStorage`, imperative third-party APIs, DOM APIs not covered by templates.
- **Avoid using effects to propagate state** (setting signal B when A changes) — use `computed` or `linkedSignal` instead. Writing signals in effects is allowed (since v19) but can cause extra cycles and bugs.
- Cleanup with `onCleanup`:

```ts
effect((onCleanup) => {
  const id = setInterval(() => console.log(this.mode()), 1000);
  onCleanup(() => clearInterval(id));
});
```

### `untracked` — read without tracking

```ts
effect(() => {
  const user = this.currentUser();                 // tracked
  untracked(() => this.logger.log(user, this.settings())); // settings not tracked
});
```

### `afterRenderEffect`

Like `effect`, but runs after rendering with phases (`earlyRead`, `write`, `mixedReadWrite`, `read`) for DOM measurement/manipulation without layout thrashing — browser only.

---

# 5. `linkedSignal` — writable derived state ⭐

A signal that **resets based on a source** but can also be **set manually**. Perfect for "selection that resets when options change".

```ts
import { linkedSignal, signal } from "@angular/core";

const shippingOptions = signal(["Standard", "Express", "Overnight"]);
const selectedOption = linkedSignal(() => shippingOptions()[0]);   // default = first option

selectedOption.set("Express");                 // user choice
shippingOptions.set(["Pickup", "Standard"]);   // source changes…
selectedOption();                              // …resets to "Pickup"
```

Keep the previous selection when still valid:

```ts
const selected = linkedSignal<string[], string>({
  source: shippingOptions,
  computation: (options, previous) =>
    previous && options.includes(previous.value) ? previous.value : options[0],
});
```

| `computed` | `linkedSignal` | `signal` + `effect` |
| --- | --- | --- |
| Read-only derived | Derived **and** writable | ❌ avoid for derived state |

---

# 6. Signal-Based Component APIs ⭐

## 6.1 Inputs: `input()`

```ts
import { Component, computed, input, booleanAttribute, numberAttribute } from "@angular/core";

@Component({ selector: "app-avatar", template: `<img [src]="src()" [width]="size()" [alt]="alt()" />` })
export class Avatar {
  readonly src = input.required<string>();                  // required
  readonly alt = input("");                                 // optional with default
  readonly size = input(48, { transform: numberAttribute }); // "64" → 64
  readonly rounded = input(false, { transform: booleanAttribute }); // <app-avatar rounded />
  readonly userName = input<string>(undefined, { alias: "name" });

  protected readonly initials = computed(() => this.userName()?.slice(0, 2).toUpperCase() ?? "");
}
```

- Inputs are **read-only signals** — derive with `computed`, react with `effect` (no `ngOnChanges` needed).
- Missing required inputs are compile-time errors.

## 6.2 Outputs: `output()`

```ts
import { output } from "@angular/core";

export class SearchBox {
  readonly search = output<string>();
  readonly cleared = output<void>();

  submit(q: string) {
    this.search.emit(q);
  }
}
```

```html
<app-search-box (search)="onSearch($event)" (cleared)="reset()" />
```

`output()` isn't an RxJS `EventEmitter` (lighter); use `outputFromObservable(obs$)` / `outputToObservable(output)` for interop.

## 6.3 Two-way: `model()`

```ts
import { model } from "@angular/core";

@Component({
  selector: "app-rating",
  template: `
    @for (star of [1, 2, 3, 4, 5]; track star) {
      <button (click)="value.set(star)" [attr.aria-pressed]="star <= value()">★</button>
    }
  `,
})
export class Rating {
  readonly value = model(0);   // writable input + implicit (valueChange) output
}
```

```html
<app-rating [(value)]="productRating" />   <!-- productRating is a signal in the parent -->
```

## 6.4 Signal queries

```ts
import { contentChildren, ElementRef, viewChild, viewChildren } from "@angular/core";

export class Gallery {
  private readonly searchInput = viewChild<ElementRef<HTMLInputElement>>("search");
  private readonly player = viewChild.required(VideoPlayer);          // by component type
  private readonly items = viewChildren(GalleryItem);                 // Signal<readonly GalleryItem[]>
  private readonly tabs = contentChildren(Tab);                       // projected content

  focusSearch() {
    this.searchInput()?.nativeElement.focus();
  }
}
```

Results are signals → `computed(() => this.items().length)` updates automatically. Replaces `@ViewChild`/`@ContentChildren` decorators and `QueryList`.

---

# 7. Signals in Templates

```html
<p>{{ count() }}</p>                         <!-- call to read -->
<button (click)="count.update(c => c + 1)">+</button>
@if (user(); as u) { <span>{{ u.name }}</span> }
```

When a signal read in a template changes, Angular marks **that component's view** for refresh — even under OnPush and in zoneless apps.

---

# 8. RxJS Interop ⭐

```ts
import { toObservable, toSignal, takeUntilDestroyed } from "@angular/core/rxjs-interop";

// Observable → Signal (subscribes immediately, unsubscribes on destroy)
readonly users = toSignal(this.http.get<User[]>("/api/users"), { initialValue: [] });
readonly params = toSignal(inject(ActivatedRoute).paramMap);

// Signal → Observable (emits asynchronously when the signal changes)
private readonly query = signal("");
readonly results = toSignal(
  toObservable(this.query).pipe(
    debounceTime(300),
    distinctUntilChanged(),
    switchMap((q) => this.api.search(q)),
  ),
  { initialValue: [] },
);
```

Options: `initialValue`, `requireSync: true` (for synchronously-emitting sources like `BehaviorSubject`), `equal`.

### `debounced()` (v22, experimental)

```ts
import { debounced } from "@angular/core";
protected readonly debouncedQuery = debounced(() => this.query(), 300);
// returns a Resource: debouncedQuery.value(), debouncedQuery.isLoading()
```

---

# 9. Async Data with Signals: Resources (preview)

```ts
readonly userId = input.required<string>();
readonly user = httpResource<User>(() => `/api/users/${this.userId()}`);  // refetches when userId changes
// user.value(), user.isLoading(), user.error(), user.reload()
```

Stable in v22 — see `09-http-and-resources.md`.

---

# 10. Signals vs Observables ⭐⭐⭐

| | Signals | Observables (RxJS) |
| --- | --- | --- |
| Model | A **value** that changes over time (always has a current value) | A **stream** of events over time |
| Read | Synchronous: `count()` | Subscribe (async) |
| Subscriptions | Automatic tracking, no unsubscribe | Manual (`subscribe`/`unsubscribe`, `async` pipe, `takeUntilDestroyed`) |
| Derived values | `computed` (lazy, memoized) | `map`, `combineLatest`… |
| Time-based operators | Few (`debounced`) | Rich (`debounceTime`, `throttle`, `retry`, `switchMap`…) |
| Best for | **UI/component state**, derived state, template binding | **Events & async workflows**: HTTP pipelines, websockets, user input streams, cancellation, complex orchestration |
| Change detection | Integrated (fine-grained) | Through `async` pipe or `toSignal` |

**Rule of thumb:** state → signals; events/async orchestration → RxJS; bridge with `toSignal`/`toObservable`.

---

# 11. Common Mistakes

| Mistake | Fix |
| --- | --- |
| Forgetting `()` when reading in templates/code | `count()` |
| Mutating objects/arrays inside a signal | Immutable `update` |
| Using `effect` to set derived state | `computed` / `linkedSignal` |
| Creating `effect` outside an injection context | Create in constructor/field, or pass `{ injector }` |
| Expecting `toObservable` to emit synchronously | It emits async, after change detection |
| Reading signals inside `untracked` expecting reactivity | Read outside `untracked` |
| Huge single signal for all state | Split into focused signals + computeds |

---

# 12. Output Questions 🧪

```ts
const a = signal(1);
const b = computed(() => a() * 2);
let runs = 0;
const c = computed(() => { runs++; return b() + 1; });

a.set(2);
a.set(3);
console.log(runs);        // Q1
console.log(c(), runs);   // Q2
console.log(c(), runs);   // Q3
a.set(3);
console.log(c(), runs);   // Q4
```

<details><summary>Answers</summary>

1. `0` — computeds are lazy; nothing has read `c`.
2. `7 1` — computed once on first read.
3. `7 1` — memoized.
4. `7 1` — setting an equal value doesn't notify.

</details>

---

# 13. Interview Answers

**What are signals in Angular?**

> "Reactive primitives that hold a value and track where they're read. `signal` creates writable state, `computed` derives memoized read-only values, and `effect` runs side effects when dependencies change. Because Angular knows which templates read which signals, it can update only affected views — that's what enables OnPush-by-default and zoneless change detection."

**computed vs effect vs linkedSignal?**

> "`computed` is for pure derived values — lazy and memoized. `effect` is for side effects that sync state to the outside world, like localStorage, logging or imperative APIs — not for deriving state. `linkedSignal` is a derived value that can also be written, resetting when its source changes — for example a selected option that resets when the options list changes."

**Signals vs RxJS — will signals replace RxJS?**

> "No, they complement each other. Signals are best for synchronous state and template binding — always a current value, automatic dependency tracking, no subscriptions. RxJS is better for event streams and async orchestration — debouncing, cancellation with switchMap, retries, websockets. I convert at the boundaries with `toSignal` and `toObservable`, and resources like `httpResource` cover many simple fetch cases without RxJS."

**What are signal inputs and why use them?**

> "`input()` declares an input as a read-only signal, `input.required()` makes it mandatory at compile time, `model()` creates a two-way bindable input, and `output()` declares events. Because inputs are signals, I can derive values with `computed` and react with `effect` instead of using `ngOnChanges`, and they integrate with fine-grained change detection."

---

# 14. Key Takeaways

- `signal` (writable), `computed` (derived, lazy, memoized), `effect` (side effects), `linkedSignal` (writable derived), `untracked`.
- Immutable updates; `Object.is` equality.
- `input()`, `input.required()`, `model()`, `output()`, signal queries replace decorators.
- `toSignal` / `toObservable` for RxJS interop; resources for async data.
- Signals = state; RxJS = streams/async orchestration.

## Readiness Checklist

- [ ] Build a cart with signals + computed totals
- [ ] Explain why effects shouldn't propagate state
- [ ] Use `linkedSignal` for dependent selection
- [ ] Convert decorator inputs/outputs to signal APIs
- [ ] Build debounced search with `toObservable` + `switchMap` + `toSignal`
