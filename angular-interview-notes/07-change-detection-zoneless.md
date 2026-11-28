# 07 — Change Detection & Zoneless 🔥🔥🔥

> **Interview + Understanding Guide** — Roadmap Section 7
>
> Goal: Explain how Angular keeps the DOM in sync with state — the component tree check, zone.js, OnPush (the default in v22) vs Eager, signals-driven fine-grained updates, zoneless Angular, and the infamous `ExpressionChangedAfterItHasBeenCheckedError`.

---

# 1. What is Change Detection? ⭐

**Change detection (CD)** is the process by which Angular **checks component templates for changes in their bindings** and updates the DOM accordingly.

```text
Something happens (click, HTTP response, timer, signal set)
   ↓
Angular is notified a change MAY have happened
   ↓
Angular walks the component tree (top-down) and re-evaluates template bindings
of components that need checking
   ↓
Changed bindings → DOM updated
```

Two questions define any CD strategy:
1. **When** does Angular run CD? (who notifies it)
2. **Which** components does it check?

---

# 2. Classic Model: zone.js ⭐

**zone.js** monkey-patches browser async APIs (`setTimeout`, `Promise`, `addEventListener`, `XMLHttpRequest`, `fetch`…). Angular runs the app inside **NgZone**; when any async task completes, the zone notifies Angular, which runs **change detection from the root**.

```text
click → zone.js intercepts → handler runs → zone: "task done" → ApplicationRef.tick()
       → check entire component tree (except skipped OnPush subtrees)
```

**Pros:** "magic" — any mutation anywhere gets picked up.
**Cons:**
- Angular doesn't know *what* changed → checks lots of components unnecessarily
- Extra bundle size (~30 KB+) and startup cost
- Noisy stack traces, harder debugging
- Patching can conflict with native async/await (requires downleveling) and third-party code
- Runs CD for irrelevant events (e.g. mouse moves handled by a library)

`NgZone.runOutsideAngular(() => …)` avoids triggering CD for high-frequency events.

---

# 3. Strategies: `OnPush` (default in v22) vs `Eager` ⭐⭐⭐

## 3.1 Eager (formerly `Default`)

The component is checked **every time** change detection runs anywhere above it.

🆕 **v22**: the old `ChangeDetectionStrategy.Default` is renamed **`Eager`**, and **`OnPush` is now the default** for components that don't specify a strategy. The `ng update` migration adds `changeDetection: ChangeDetectionStrategy.Eager` to existing components without an explicit strategy so their behaviour doesn't change.

## 3.2 OnPush

An OnPush component (and its subtree) is checked **only when it's marked dirty**, which happens when:

1. An **input reference changes** (`Object.is` comparison) — mutating an input object doesn't count ⚠️
2. An **event** handled in its template (or a host listener) fires
3. A **signal read in its template** changes
4. The **`async` pipe** in its template receives a new value
5. It's manually marked with `ChangeDetectorRef.markForCheck()`

When a component is marked dirty, its **ancestors** are marked too (so the path from root gets checked).

```ts
@Component({
  selector: "app-user-list",
  changeDetection: ChangeDetectionStrategy.OnPush,   // implicit in v22
  template: `@for (u of users(); track u.id) { <app-user-row [user]="u" /> }`,
})
export class UserList {
  readonly users = input.required<User[]>();
}
```

### ⚠️ The classic OnPush bug

```ts
// parent
addUser(user: User) {
  this.users.push(user);            // ❌ same array reference → OnPush child not updated
  this.users = [...this.users, user]; // ✅ new reference → child updated
}
```

### Benefits of OnPush

- Skips whole subtrees during CD → big performance wins in large trees
- Encourages **immutable data** and **unidirectional flow**
- Predictable rendering

## 3.3 Manual control: `ChangeDetectorRef`

```ts
private readonly cdr = inject(ChangeDetectorRef);

this.cdr.markForCheck();   // mark this view + ancestors dirty → checked in the next CD cycle (use with OnPush)
this.cdr.detectChanges();  // run CD synchronously for this view and its children now
this.cdr.detach();         // remove from CD tree (manual control, e.g. high-frequency widgets)
this.cdr.reattach();
```

`markForCheck` is the common fix when an OnPush component updates from a non-tracked source (e.g. a callback from a third-party library, a manual subscription). With signals you rarely need it.

---

# 4. Signals & Fine-Grained Change Detection ⭐

When a signal **read in a template** changes, Angular:
1. Marks the **consumer view** (the component whose template read it) for refresh
2. Schedules change detection
3. Refreshes **that view** — ancestors are traversed but not re-checked unless dirty themselves (since v17, "local change detection" for signals in OnPush trees)

```ts
@Component({
  selector: "app-counter",
  template: `<button (click)="count.update(c => c + 1)">{{ count() }}</button>`,
})
export class Counter {
  protected readonly count = signal(0);
}
```

No `markForCheck`, no zone.js needed: the signal knows exactly which view depends on it.

---

# 5. Zoneless Angular ⭐⭐⭐

**Zoneless** = no zone.js. Angular schedules change detection only when it's **explicitly notified**:

| Notification source | Triggers CD |
| --- | --- |
| Signal read in a template changes | ✅ |
| Template/host event listener runs | ✅ |
| `markForCheck()` (incl. via `async` pipe) | ✅ |
| `ComponentRef.setInput()` | ✅ |
| View attached/detached | ✅ |
| `setTimeout`, `Promise.then`, `fetch` resolving **without** updating a signal/`markForCheck` | ❌ nothing happens |

### Status by version

| Version | Zoneless |
| --- | --- |
| v18 | Experimental (`provideExperimentalZonelessChangeDetection`) |
| v20 | Developer preview → stable in v20.2 (`provideZonelessChangeDetection()`) |
| v21 | **Default for new apps** — zone.js no longer included |
| v22 | Default + OnPush default |

```ts
// Older apps opting in (v20.2+): remove zone.js from polyfills and add
providers: [provideZonelessChangeDetection()]

// New apps (v21+) that need zone.js back:
providers: [provideZoneChangeDetection({ eventCoalescing: true })]   // + "zone.js" polyfill
```

### Making code zoneless-compatible

```ts
// ❌ Works with zone.js, but NOT zoneless: plain field mutated in a timer
export class Clock {
  time = new Date();
  constructor() {
    setInterval(() => (this.time = new Date()), 1000);   // no notification → view never updates
  }
}

// ✅ Use a signal
export class ClockFixed {
  protected readonly time = signal(new Date());
  constructor() {
    const id = setInterval(() => this.time.set(new Date()), 1000);
    inject(DestroyRef).onDestroy(() => clearInterval(id));
  }
}
```

Checklist: state in **signals**, `async` pipe/`toSignal` for observables, OnPush-compatible components, no reliance on `NgZone.onStable`/`onMicrotaskEmpty`, use `afterNextRender` instead of zone timing hacks. `PendingTasks` tells SSR when async work is in progress.

### Benefits

- Smaller bundles, faster startup
- Change detection only when needed
- Clean stack traces; native `async/await`
- Better interop with other frameworks/micro-frontends
- Predictable: changes come from signals/events

---

# 6. `ExpressionChangedAfterItHasBeenCheckedError` ⭐⭐⭐

In **development mode**, Angular runs an extra verification pass after each CD cycle. If a binding's value **changed after it was checked** in the same cycle, Angular throws:

```text
NG0100: ExpressionChangedAfterItHasBeenCheckedError:
Expression has changed after it was checked. Previous value: 'false'. Current value: 'true'.
```

It signals a **unidirectional data flow violation**: something updated state that a parent (or the same view) already rendered.

### Common causes

```ts
// 1. Updating parent state from a child's ngAfterViewInit / ngOnInit
export class Child implements AfterViewInit {
  private readonly parentState = inject(PageState);
  ngAfterViewInit() {
    this.parentState.title = "Loaded";     // parent already checked its {{ title }} binding
  }
}

// 2. Getter returning a new value every time
get now() { return Date.now(); }              // {{ now }} differs between check & verification

// 3. Random values / new object literals in bindings
// [config]="{ a: 1 }" with checks comparing references
```

### Fixes

- Move the state update **earlier** (`ngOnInit` in the parent, or constructor) or into an event handler
- Derive values with `computed` instead of setting them in lifecycle hooks
- Use **signals** (updates schedule a fresh render instead of mutating mid-check)
- Make getters stable/pure
- As a last resort, defer to the next tick (`queueMicrotask`/`setTimeout`) or call `detectChanges()` in the child — these hide design problems

> It only throws in dev mode — but it points to real bugs (inconsistent UI) that would be silent in production.

---

# 7. Change Detection Flow (detailed)

```text
ApplicationRef.tick()
  └─ for each root view: refreshView()
       ├─ run effects created in the component (before view refresh)
       ├─ update template bindings (property/text) for this view
       ├─ run lifecycle hooks: ngDoCheck, ngAfterContentChecked…
       ├─ refresh child views that are Eager OR dirty OR have dirty signal consumers
       └─ ngAfterViewChecked
  └─ (dev mode) checkNoChanges pass → NG0100 if values differ
  └─ afterEveryRender / afterNextRender callbacks
```

---

# 8. Performance Tips Related to CD

- Keep **OnPush** (default) + **signals** for state.
- Use **immutable** updates.
- **`track`** in `@for`.
- **Pure pipes** instead of method calls in templates (`{{ format(x) }}` runs every check).
- Avoid heavy getters in templates; use `computed`.
- `runOutsideAngular` (zone apps) for scroll/mousemove/animation loops; or simply don't update signals at high frequency.
- Profile with **Angular DevTools → Profiler** (shows CD cycles, what triggered them, time per component) and the signal graph view.

---

# 9. Output / Behaviour Questions 🧪

**Q1.** OnPush child receives `[items]="items"`; parent does `items.push(x)` in a click handler. Child updates? → **No** (same reference). Parent's own template updates (event triggered CD on the parent).

**Q2.** OnPush component sets a plain field inside `setTimeout` (zone.js app). View updates? → **No**, OnPush isn't marked dirty by timers. Use a signal or `markForCheck()`.

**Q3.** Same but the component is Eager in a zone.js app? → **Yes** — zone triggers tick and Eager components are always checked.

**Q4.** Zoneless app, Eager component, plain field updated in `setTimeout`. → **No** — nothing notified Angular.

**Q5.** OnPush component, signal updated in `setTimeout`. → **Yes** — signal changes mark the view dirty in both zone and zoneless apps.

---

# 10. Interview Answers

**How does change detection work in Angular?**

> "Angular represents the UI as a tree of component views. When it's notified that something may have changed, it walks the tree top-down and re-evaluates template bindings, updating the DOM where values differ. Historically zone.js patched async APIs to notify Angular after every event, timer or HTTP call. Modern Angular uses signals: when a signal read in a template changes, Angular marks just that view dirty and schedules change detection, which enables zoneless apps — the default for new projects since v21."

**What is OnPush?**

> "A strategy where a component is only checked when it's marked dirty: an input reference changes, an event fires in its template, a signal it reads changes, the async pipe emits, or `markForCheck` is called. Otherwise its whole subtree is skipped. It requires immutable data — mutating an input object won't trigger an update. In Angular 22 OnPush is the default; the old default strategy is called Eager."

**What is zoneless change detection and why move to it?**

> "Running Angular without zone.js. Instead of patching every async API, Angular runs change detection only when explicitly notified — signal changes, template events, `markForCheck`, the async pipe, `setInput`. It removes zone.js's bundle size and overhead, avoids unnecessary checks, gives cleaner stack traces and native async/await. The requirement is that state changes go through signals or other notifying APIs."

**What causes `ExpressionChangedAfterItHasBeenCheckedError`?**

> "In dev mode Angular double-checks bindings after each cycle. The error means a value changed after it was already checked in the same cycle — usually a child updating parent state in `ngAfterViewInit`, or a getter returning a new value each time. It's a unidirectional data flow violation; I fix it by moving the update earlier or into an event, deriving values with `computed`, or using signals — not by wrapping in `setTimeout`."

**`markForCheck` vs `detectChanges`?**

> "`markForCheck` marks the view and its ancestors as dirty so they're checked in the next cycle — the normal way to tell an OnPush component it needs updating. `detectChanges` runs change detection synchronously on that view and its children immediately — useful for detached views or tests, but it can cause extra work if overused."

---

# 11. Key Takeaways

- CD re-evaluates template bindings top-down when Angular is notified.
- zone.js = automatic notifications for all async work (simple but coarse).
- OnPush (default v22) checks only dirty components; immutable inputs; Eager = old Default.
- Signals mark exactly the consuming view dirty → fine-grained, zoneless-ready.
- Zoneless is default for new apps since v21; state must notify (signals, events, async pipe).
- `NG0100` = state changed during the check → fix the data flow.

## Readiness Checklist

- [ ] Explain when an OnPush component is checked (5 triggers)
- [ ] Demonstrate the mutation bug and fix
- [ ] Convert a zone-dependent component to zoneless with signals
- [ ] Explain and fix NG0100
- [ ] Use Angular DevTools Profiler
