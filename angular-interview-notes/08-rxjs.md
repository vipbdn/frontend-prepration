# 08 — RxJS in Angular 🔥🔥

> **Interview + Understanding Guide** — Roadmap Section 8
>
> Goal: Be fluent in the RxJS that Angular interviews actually test — Observables vs Promises, Subjects, hot vs cold, the four flattening operators, error handling, sharing, unsubscribing, and the classic typeahead.

---

# 1. Observable Basics ⭐

An **Observable** is a lazy stream that can emit **0..N values over time**, then complete or error.

```ts
import { Observable } from "rxjs";

const numbers$ = new Observable<number>((subscriber) => {
  subscriber.next(1);
  subscriber.next(2);
  const id = setTimeout(() => {
    subscriber.next(3);
    subscriber.complete();
  }, 1000);
  return () => clearTimeout(id);          // teardown on unsubscribe
});

const sub = numbers$.subscribe({
  next: (v) => console.log(v),
  error: (e) => console.error(e),
  complete: () => console.log("done"),
});
sub.unsubscribe();
```

Conventions: `$` suffix for observables; Angular uses them for `HttpClient`, `Router.events`, `ActivatedRoute.paramMap`, `FormControl.valueChanges`, `EventEmitter`.

## Observable vs Promise ⭐⭐⭐

| | Promise | Observable |
| --- | --- | --- |
| Values | Exactly one | 0..N over time |
| Eager/lazy | **Eager** — runs on creation | **Lazy** — runs on subscribe |
| Cancellable | No | Yes (`unsubscribe`, `switchMap`) |
| Operators | `then`/`catch` | 100+ operators (`map`, `filter`, `retry`, `debounceTime`…) |
| Multicast | Shared result | Unicast by default (each subscriber re-runs; cold) |
| Sync/async | Always async | Can emit synchronously |

`firstValueFrom(obs$)` / `lastValueFrom(obs$)` convert to Promises; `from(promise)` converts back.

---

# 2. Creation Functions

```ts
import { of, from, interval, timer, fromEvent, EMPTY, throwError, combineLatest, forkJoin, merge, concat } from "rxjs";

of(1, 2, 3);                               // emits values then completes
from([1, 2, 3]); from(fetch("/api"));       // array / promise / iterable
interval(1000);                            // 0,1,2… every second
timer(2000);                               // one value after 2s
fromEvent<MouseEvent>(document, "click");
throwError(() => new Error("boom"));
```

### Combination ⭐

| Function | Emits | Typical use |
| --- | --- | --- |
| `forkJoin([a$, b$])` | Once, **last values** when **all complete** | Parallel HTTP calls (like `Promise.all`) |
| `combineLatest([a$, b$])` | Latest of each whenever **any** emits (after all emitted once) | Filters + data, derived view state |
| `merge(a$, b$)` | Every value from any, interleaved | Combine event sources |
| `concat(a$, b$)` | `a$` fully, then `b$` | Sequential |
| `withLatestFrom(b$)` (operator) | On source emit, pair with latest `b$` | Click + current state |
| `zip(a$, b$)` | Pairs by index | Rare |

⚠️ `forkJoin` with a never-completing source (e.g. `valueChanges`) never emits.

---

# 3. Core Operators

```ts
import { map, filter, tap, take, takeUntil, debounceTime, distinctUntilChanged, startWith, scan, catchError, retry, finalize } from "rxjs";

this.form.controls.search.valueChanges.pipe(
  map((v) => v?.trim() ?? ""),
  filter((v) => v.length > 1),
  debounceTime(300),
  distinctUntilChanged(),
  tap((v) => console.log("searching", v)),
);
```

| Operator | Purpose |
| --- | --- |
| `map` / `filter` | Transform / skip |
| `tap` | Side effects (logging) without changing values |
| `debounceTime(ms)` | Emit after silence (search input) |
| `throttleTime(ms)` | At most one per window (scroll, resize) |
| `distinctUntilChanged()` | Skip consecutive duplicates |
| `take(n)` / `first()` | Complete after n / first value |
| `takeUntil(notifier$)` | Complete when notifier emits |
| `startWith(x)` | Initial value |
| `scan(fn, seed)` | Running accumulation (reducer over time) |
| `catchError` / `retry` | Error handling |
| `finalize` | On complete/error/unsubscribe (hide spinner) |
| `shareReplay` | Multicast + cache |

---

# 4. Higher-Order Mapping — the Big Four ⭐⭐⭐

When each value triggers **another observable** (usually HTTP), choose how inner observables are combined:

| Operator | Behaviour | Use case |
| --- | --- | --- |
| **`switchMap`** | **Cancels** the previous inner, switches to the new one | Search/typeahead, route param → load data (only latest matters) |
| **`mergeMap`** (`flatMap`) | Runs all **in parallel**, results in completion order | Independent parallel requests (deleting many items), fire-and-forget |
| **`concatMap`** | **Queues** — one at a time, in order | Ordered writes/saves, sequential uploads |
| **`exhaustMap`** | **Ignores** new values while the current inner is active | Login/submit button (prevent double submission), refresh |

```ts
// Typeahead: abort stale requests
query$.pipe(switchMap((q) => this.api.search(q)));

// Save each change in order
changes$.pipe(concatMap((c) => this.api.save(c)));

// Ignore double-clicks while submitting
submit$.pipe(exhaustMap(() => this.api.login(this.form.getRawValue())));

// Parallel with concurrency limit
ids$.pipe(mergeMap((id) => this.api.delete(id), 3));
```

Memory trick: **switch** = latest wins, **merge** = all at once, **concat** = queue, **exhaust** = first wins.

---

# 5. Subjects ⭐⭐

A **Subject** is both an Observable and an Observer — you can `next()` values into it, and it **multicasts** to all subscribers.

| Type | New subscriber receives | Use |
| --- | --- | --- |
| `Subject` | Only future values | Event bus, notifiers (`destroy$`) |
| `BehaviorSubject(init)` | **Current value** immediately, then future | State (has `.value`) |
| `ReplaySubject(n)` | Last **n** values | Cache recent events |
| `AsyncSubject` | Last value, only on complete | Rare |

```ts
const s = new Subject<number>();
s.next(1);                       // lost — no subscribers yet
s.subscribe((v) => console.log("A", v));
s.next(2);                       // A 2

const b = new BehaviorSubject(0);
b.next(1);
b.subscribe((v) => console.log("B", v));   // B 1 (current value)
```

Expose read-only: `readonly state$ = this.subject.asObservable();`

---

# 6. Hot vs Cold & Sharing ⭐⭐

- **Cold**: the producer is created **per subscriber** (`HttpClient.get`, `of`, `interval`). Two subscriptions → **two HTTP requests**.
- **Hot**: the producer exists independently; subscribers share it (`Subject`, DOM events, websockets).

```ts
// ⚠️ two requests
const users$ = this.http.get<User[]>("/api/users");
users$.subscribe(); users$.subscribe();

// template with two async pipes on the same cold observable → two requests too!
```

```ts
// ✅ share + cache last value
readonly users$ = this.http.get<User[]>("/api/users").pipe(shareReplay({ bufferSize: 1, refCount: true }));
```

- `share()` — multicast while there are subscribers; late subscribers miss earlier values.
- `shareReplay({ bufferSize: 1, refCount: true })` — also replays the last value; `refCount: true` unsubscribes from source when nobody listens (avoids leaks for infinite sources).
- Or use `toSignal()` once and read the signal in many places.

---

# 7. Error Handling ⭐

An error **terminates** the stream. In long-lived streams (e.g. typeahead), catch errors **inside** the inner observable so the outer keeps working:

```ts
query$.pipe(
  switchMap((q) =>
    this.api.search(q).pipe(
      retry({ count: 2, delay: 1000 }),
      catchError((err) => {
        this.error.set("Search failed");
        return of([]);            // fallback, outer stream continues
      }),
    ),
  ),
);
```

⚠️ `catchError` on the outer stream replaces it and the stream completes — later queries do nothing.

---

# 8. Unsubscribing & Memory Leaks ⭐⭐⭐

Finite streams (`HttpClient` calls) complete on their own. **Infinite** ones (`interval`, `valueChanges`, `router.events`, store selectors, websockets) must be cleaned up.

| Approach | Notes |
| --- | --- |
| **`async` pipe** | Subscribes/unsubscribes automatically, marks OnPush dirty ✅ |
| **`toSignal()`** | Auto-unsubscribes on destroy ✅ |
| **`takeUntilDestroyed()`** | Operator using `DestroyRef` ✅ |
| `takeUntil(this.destroy$)` + `ngOnDestroy` | Classic pattern; put it **last** in the pipe |
| Manual `Subscription.unsubscribe()` | Verbose, error-prone |

```ts
import { takeUntilDestroyed } from "@angular/core/rxjs-interop";

export class Clock {
  constructor() {
    interval(1000).pipe(takeUntilDestroyed()).subscribe(() => this.tick());   // in injection context
  }

  private readonly destroyRef = inject(DestroyRef);
  later() {
    interval(1000).pipe(takeUntilDestroyed(this.destroyRef)).subscribe();    // outside: pass DestroyRef
  }
}
```

Avoid **nested subscribes** — use flattening operators.

---

# 9. Classic: Typeahead Search ⭐⭐⭐

```ts
@Component({
  selector: "app-user-search",
  imports: [ReactiveFormsModule],
  template: `
    <input [formControl]="query" placeholder="Search users" />
    @if (loading()) { <p>Loading…</p> }
    <ul>
      @for (u of results(); track u.id) { <li>{{ u.name }}</li> } @empty { <li>No results</li> }
    </ul>
  `,
})
export class UserSearch {
  private readonly api = inject(UserApi);
  protected readonly query = new FormControl("", { nonNullable: true });
  protected readonly loading = signal(false);

  protected readonly results = toSignal(
    this.query.valueChanges.pipe(
      map((q) => q.trim()),
      debounceTime(300),
      distinctUntilChanged(),
      tap(() => this.loading.set(true)),
      switchMap((q) =>
        (q.length < 2 ? of([]) : this.api.search(q)).pipe(
          catchError(() => of([])),
          finalize(() => this.loading.set(false)),
        ),
      ),
    ),
    { initialValue: [] },
  );
}
```

Why each piece: `debounceTime` reduces requests, `distinctUntilChanged` skips repeats, `switchMap` cancels stale requests (prevents race conditions), inner `catchError` keeps the stream alive, `toSignal` handles subscription.

---

# 10. Other Angular Patterns

```ts
// Route param → data, cancelling on param change
readonly product = toSignal(
  inject(ActivatedRoute).paramMap.pipe(
    map((p) => p.get("id")!),
    switchMap((id) => this.api.getProduct(id)),
  ),
);

// Polling
readonly status$ = timer(0, 5000).pipe(switchMap(() => this.api.status()));

// Router events
inject(Router).events.pipe(filter((e): e is NavigationEnd => e instanceof NavigationEnd), takeUntilDestroyed());
```

---

# 11. Output Questions 🧪

```ts
const s = new BehaviorSubject(1);
s.next(2);
s.subscribe((v) => console.log("A", v));
s.next(3);
s.subscribe((v) => console.log("B", v));
```

<details><summary>Answer</summary>

`A 2`, `A 3`, `B 3` — BehaviorSubject gives each new subscriber the current value.

</details>

```ts
of(1, 2, 3).pipe(switchMap((x) => timer(100).pipe(map(() => x)))).subscribe(console.log);
```

<details><summary>Answer</summary>

`3` — `of` emits synchronously, each new value cancels the previous timer; only the last survives. With `mergeMap` → `1 2 3`; `concatMap` → `1 2 3` (sequentially, 300ms total); `exhaustMap` → `1`.

</details>

```ts
console.log("a");
of("b").subscribe(console.log);
Promise.resolve("c").then(console.log);
console.log("d");
```

<details><summary>Answer</summary>

`a b d c` — `of` emits synchronously; the promise callback is a microtask.

</details>

---

# 12. Interview Answers

**Observable vs Promise?**

> "A promise is eager, resolves once and can't be cancelled. An observable is lazy — nothing happens until you subscribe — can emit many values over time, can be cancelled by unsubscribing, and has operators for transforming, combining, retrying and debouncing. Angular uses observables for HTTP, router events and form value changes."

**switchMap vs mergeMap vs concatMap vs exhaustMap?**

> "All four map each value to an inner observable. `switchMap` cancels the previous inner — ideal for search. `mergeMap` runs them concurrently — independent parallel requests. `concatMap` queues them in order — sequential saves. `exhaustMap` ignores new values while one is running — preventing double submits."

**Subject vs BehaviorSubject?**

> "Both multicast and can be pushed into with `next`. A Subject only emits future values. A BehaviorSubject needs an initial value, stores the current value, exposes it via `.value`, and emits it immediately to new subscribers — which is why it was the classic choice for service state."

**How do you avoid memory leaks with subscriptions?**

> "Prefer the async pipe or `toSignal`, which clean up automatically. For manual subscriptions use `takeUntilDestroyed`, or `takeUntil` with a destroy subject as the last operator. HTTP observables complete on their own, but infinite streams like intervals, valueChanges and router events must be torn down. And avoid nested subscribes in favour of flattening operators."

**Hot vs cold observables?**

> "A cold observable creates its producer per subscriber — every subscription to an HttpClient call sends a new request. A hot one shares an existing producer, like a Subject or DOM events. To share a cold observable I use `share` or `shareReplay({bufferSize: 1, refCount: true})`, or convert it once with `toSignal`."

---

# 13. Key Takeaways

- Observables: lazy, multi-value, cancellable.
- Know `forkJoin` vs `combineLatest`; the big four flattening operators.
- Subjects multicast; BehaviorSubject holds state.
- Cold HTTP → share with `shareReplay` or `toSignal`.
- Catch errors in inner streams; clean up infinite streams.
- Signals for state, RxJS for async/event orchestration.

## Readiness Checklist

- [ ] Explain the four flattening operators with use cases
- [ ] Build typeahead with debounce + switchMap + catchError
- [ ] Explain hot vs cold and fix a double-request bug
- [ ] List 4 ways to unsubscribe
