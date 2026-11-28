# 05 — Component Communication 🔥

> **Interview + Understanding Guide** — Roadmap Section 5
>
> Goal: Know every way components share data — inputs, outputs, two-way binding, template refs and queries, shared services with signals, content projection, router state — and pick the right one.

---

# 1. Overview ⭐

| Relationship | Technique |
| --- | --- |
| Parent → Child | **Inputs** (`input()` / `@Input`) |
| Child → Parent | **Outputs** (`output()` / `@Output` + `EventEmitter`) |
| Two-way parent ↔ child | **`model()`** with `[(x)]` |
| Parent calls child methods / reads child state | Template reference variable, `viewChild()` |
| Siblings / distant components | **Shared service** with signals (or RxJS Subjects) |
| Parent passes markup | Content projection (`ng-content`), `ng-template` inputs |
| Across routes | Route params / query params / router state / service |
| App-wide state | Service store, NgRx (see `13-state-management.md`) |
| Ancestor provides to descendants | DI at component level (`providers`) — context-like |

---

# 2. Parent → Child: Inputs ⭐

```ts
// child: user-card.ts
@Component({
  selector: "app-user-card",
  template: `<h3>{{ user().name }}</h3><p>{{ role() }}</p>`,
})
export class UserCard {
  readonly user = input.required<User>();
  readonly role = input<"admin" | "member">("member");
}
```

```html
<!-- parent template -->
<app-user-card [user]="selectedUser()" role="admin" />
```

### Decorator style (legacy, still very common)

```ts
export class UserCardLegacy implements OnChanges {
  @Input({ required: true }) user!: User;
  @Input() role: "admin" | "member" = "member";

  // react to changes
  ngOnChanges(changes: SimpleChanges) {
    if (changes["user"] && !changes["user"].firstChange) {
      console.log("user changed", changes["user"].previousValue, changes["user"].currentValue);
    }
  }
}
```

Setter alternative: `@Input() set user(value: User) { … }`.

⚠️ Inputs should be treated as **read-only** in the child. Don't mutate input objects — emit events instead.

---

# 3. Child → Parent: Outputs ⭐

```ts
@Component({
  selector: "app-todo-item",
  template: `
    <input type="checkbox" [checked]="todo().done" (change)="toggled.emit(todo().id)" />
    {{ todo().title }}
    <button (click)="removed.emit(todo().id)">✕</button>
  `,
})
export class TodoItem {
  readonly todo = input.required<Todo>();
  readonly toggled = output<number>();
  readonly removed = output<number>();
}
```

```html
@for (t of todos(); track t.id) {
  <app-todo-item [todo]="t" (toggled)="toggle($event)" (removed)="remove($event)" />
}
```

Legacy: `@Output() removed = new EventEmitter<number>();`

---

# 4. Two-Way Binding with `model()` ⭐

```ts
@Component({
  selector: "app-quantity",
  template: `
    <button (click)="qty.update(q => Math.max(1, q - 1))">−</button>
    <span>{{ qty() }}</span>
    <button (click)="qty.update(q => q + 1)">+</button>
  `,
})
export class Quantity {
  readonly qty = model(1);
  protected readonly Math = Math;
}
```

```html
<app-quantity [(qty)]="quantity" />      <!-- quantity = signal(1) in parent -->
<p>Total: {{ quantity() * price }}</p>
```

`[(qty)]` = `[qty]="quantity()"` + `(qtyChange)="quantity.set($event)"`.

Legacy pattern: `@Input() qty` + `@Output() qtyChange = new EventEmitter<number>()`.

---

# 5. Parent Accessing the Child: Template Refs & `viewChild` ⭐

```html
<!-- template reference variable -->
<app-stopwatch #sw />
<button (click)="sw.start()">Start</button>
<button (click)="sw.reset()">Reset</button>
```

```ts
// in the parent class
export class Dashboard {
  private readonly stopwatch = viewChild.required(Stopwatch);

  resetAll() {
    this.stopwatch().reset();
  }
}
```

Use sparingly — prefer inputs/outputs for declarative data flow. Imperative access is fine for focus, media playback, starting animations, etc.

---

# 6. Shared Service with Signals ⭐⭐⭐ (siblings / any components)

```ts
// cart.store.ts
import { computed, signal } from "@angular/core";
import { Service } from "@angular/core";        // v22; or @Injectable({ providedIn: "root" })

export interface CartItem { id: string; name: string; price: number; qty: number }

@Service()
export class CartStore {
  private readonly _items = signal<CartItem[]>([]);

  readonly items = this._items.asReadonly();
  readonly count = computed(() => this._items().reduce((n, i) => n + i.qty, 0));
  readonly total = computed(() => this._items().reduce((s, i) => s + i.price * i.qty, 0));

  add(product: Omit<CartItem, "qty">) {
    this._items.update((items) => {
      const existing = items.find((i) => i.id === product.id);
      return existing
        ? items.map((i) => (i.id === product.id ? { ...i, qty: i.qty + 1 } : i))
        : [...items, { ...product, qty: 1 }];
    });
  }

  remove(id: string) {
    this._items.update((items) => items.filter((i) => i.id !== id));
  }
}
```

```ts
// header.ts — reads
@Component({ selector: "app-header", template: `🛒 {{ cart.count() }}` })
export class Header {
  protected readonly cart = inject(CartStore);
}

// product-card.ts — writes
@Component({ selector: "app-product", template: `<button (click)="cart.add(product())">Add</button>` })
export class ProductCardBtn {
  readonly product = input.required<Omit<CartItem, "qty">>();
  protected readonly cart = inject(CartStore);
}
```

- Private writable signal + public read-only signals + methods = **encapsulated mini-store**.
- Scope: `providedIn: "root"` / `@Service()` → app singleton; or `providers: [CartStore]` on a component → one instance per component subtree.

### RxJS version (classic, very common in interviews)

```ts
@Injectable({ providedIn: "root" })
export class CartService {
  private readonly itemsSubject = new BehaviorSubject<CartItem[]>([]);
  readonly items$ = this.itemsSubject.asObservable();
  readonly count$ = this.items$.pipe(map((items) => items.reduce((n, i) => n + i.qty, 0)));

  add(item: CartItem) {
    this.itemsSubject.next([...this.itemsSubject.value, item]);
  }
}
```

```html
🛒 {{ cart.count$ | async }}
```

`BehaviorSubject` holds a current value and replays it to new subscribers — the pre-signals way to share state.

---

# 7. Content Projection & Template Inputs

The parent controls what the child renders:

```html
<app-modal [open]="showModal()" (closed)="showModal.set(false)">
  <h2 modal-title>Delete item?</h2>
  <p>This can't be undone.</p>
</app-modal>
```

See `02-components-templates-binding.md` §6 and `03-control-flow-and-defer.md` §3.

---

# 8. Communication via the Router

```ts
// Route: { path: "products/:id", component: ProductDetail }
// app.config: provideRouter(routes, withComponentInputBinding())

export class ProductDetail {
  readonly id = input.required<string>();     // bound from :id automatically
  readonly tab = input<string>();             // from ?tab=reviews (query param)
}
```

```ts
// Programmatic
this.router.navigate(["/products", id], { queryParams: { tab: "reviews" }, state: { from: "search" } });
```

Use URL params for shareable state (filters, selected tab); `history.state` for transient data; services for complex data.

---

# 9. Component-Level DI (context-like)

```ts
@Component({
  selector: "app-wizard",
  providers: [WizardState],          // new instance for this wizard and its descendants
  template: `<app-step-one /><app-step-two />`,
})
export class Wizard {}

@Component({ selector: "app-step-one", template: `…` })
export class StepOne {
  private readonly state = inject(WizardState);   // gets the wizard's instance
}
```

Equivalent to React Context scoped to a subtree. See `06-dependency-injection.md`.

---

# 10. Choosing ⭐

```text
Direct parent/child?            → inputs / outputs / model
Parent needs to call child?     → template ref / viewChild (sparingly)
Siblings or unrelated?          → shared service (signals)
Scoped to a feature subtree?    → component-level provider
Shareable via URL?              → route/query params
Large app-wide state w/ tooling? → NgRx / SignalStore
```

---

# 11. Interview Answers

**How do components communicate in Angular?**

> "Parent to child with inputs, child to parent with outputs, and two-way with `model()` and the banana-in-a-box syntax. A parent can reach a child through a template reference variable or `viewChild`. For siblings or distant components I use a shared service — today usually signals: a private writable signal, public read-only signals and computeds, and methods to update — or a `BehaviorSubject` in older code. Route parameters and content projection cover the rest, and NgRx for large shared state."

**What is `EventEmitter` and how does `output()` differ?**

> "`EventEmitter` extends RxJS `Subject` and was used with `@Output` to emit events to the parent. `output()` is the modern function-based API — lighter, not an RxJS subject, and type-safe — with interop helpers for observables."

**Why use a `BehaviorSubject` for shared state?**

> "Because it holds the current value and immediately emits it to new subscribers, so late subscribers get the latest state — unlike a plain Subject. With signals, a writable signal plays that role more simply."

---

# 12. Key Takeaways

- Inputs down, outputs up, `model()` for two-way.
- Template refs/`viewChild` for imperative needs only.
- Shared services with private writable + public read-only signals for siblings/global state.
- Component-level providers scope state to a subtree.
- Router params for URL-shareable state.

## Readiness Checklist

- [ ] Build parent/child with input, output and model
- [ ] Build a signal-based cart store used by two sibling components
- [ ] Rewrite it with a BehaviorSubject and async pipe
- [ ] Bind route params to inputs with `withComponentInputBinding`
