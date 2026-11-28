# 02 — Components, Templates & Data Binding 🔥

> **Interview + Understanding Guide** — Roadmap Section 2
>
> Goal: Master component anatomy, every kind of data binding, template features, lifecycle hooks (and the newer render hooks), view encapsulation and content projection.

---

# 1. Component Anatomy ⭐

```ts
import { ChangeDetectionStrategy, Component, computed, input, output, signal } from "@angular/core";
import { CurrencyPipe } from "@angular/common";

@Component({
  selector: "app-product-card",                 // how it's used: <app-product-card />
  imports: [CurrencyPipe],                      // template dependencies (standalone)
  templateUrl: "./product-card.html",           // or inline `template`
  styleUrl: "./product-card.css",               // or `styles`
  changeDetection: ChangeDetectionStrategy.OnPush, // default in v22 (explicit is fine)
  host: { class: "card", "[class.featured]": "featured()" },  // host element bindings
})
export class ProductCard {
  // Inputs / outputs (signal-based)
  readonly product = input.required<Product>();
  readonly featured = input(false);
  readonly addToCart = output<Product>();

  // Local state
  protected readonly quantity = signal(1);
  protected readonly total = computed(() => this.product().price * this.quantity());

  protected add() {
    this.addToCart.emit(this.product());
  }
}

export interface Product {
  id: string;
  name: string;
  price: number;
}
```

```html
<!-- product-card.html -->
<h3>{{ product().name }}</h3>
<p>{{ total() | currency: "INR" }}</p>
<button (click)="quantity.update(q => q + 1)">+</button>
<button (click)="add()">Add to cart</button>
```

### `@Component` metadata

| Property | Purpose |
| --- | --- |
| `selector` | CSS selector: element `app-x`, attribute `[appX]`, class `.x` |
| `template` / `templateUrl` | The view |
| `styles` / `styleUrl(s)` | Component-scoped CSS |
| `imports` | Components, directives, pipes used in the template |
| `changeDetection` | `OnPush` (default v22) or `Eager` |
| `host` | Bindings/listeners on the host element |
| `providers` / `viewProviders` | Element-level DI providers |
| `encapsulation` | `Emulated` (default), `ShadowDom`, `None` |
| `hostDirectives` | Compose directives onto the host |
| `exportAs` | Name for template reference (`#tabs="tabs"`) |

💡 Selector prefix (`app-`, or your project prefix) avoids clashes with native/custom elements.

💡 Use `protected` for members used only by the template, `readonly` for signals/inputs.

---

# 2. Data Binding ⭐⭐⭐

```text
Component ──────────────▶ Template     {{ value }}  [property]="expr"  [attr.x]  [class.x]  [style.x]
Component ◀────────────── Template     (event)="handler($event)"
Component ◀─────────────▶ Template     [(ngModel)]="value"  [(value)]="sig"  (two-way)
```

| Type | Syntax | Example |
| --- | --- | --- |
| **Interpolation** | `{{ expr }}` | `<h1>{{ user().name }}</h1>` |
| **Property binding** | `[prop]="expr"` | `<img [src]="imageUrl()" [alt]="name()">` |
| **Attribute binding** | `[attr.name]="expr"` | `<td [attr.colspan]="span">`, `[attr.aria-label]="label"` |
| **Class binding** | `[class.name]="bool"` / `[class]="expr"` | `<li [class.active]="isActive()">` |
| **Style binding** | `[style.prop]="expr"` | `<div [style.width.px]="width()">` |
| **Event binding** | `(event)="stmt"` | `<button (click)="save()">`, `(keyup.enter)="search()"` |
| **Two-way binding** | `[(prop)]="sig"` | `<input [(ngModel)]="name">`, `<app-rating [(value)]="rating">` |

### Property vs attribute

- **Attributes** are what's in the HTML (initial values, strings).
- **Properties** are live DOM object values. Angular binds to **properties** by default.
- Use `[attr.x]` when there's no matching property (`colspan`, `aria-*`, `data-*`, SVG attributes).

### Class & style bindings

```html
<div [class.error]="hasError()"></div>
<div [class]="{ active: isActive(), disabled: isDisabled() }"></div>
<div [class]="['btn', variant()]"></div>
<div [style.background-color]="color()" [style.width.%]="progress()"></div>
<div [style]="{ 'font-size.px': size(), color: color() }"></div>
```

(`NgClass`/`NgStyle` still work but plain bindings are preferred.)

### Events

```html
<input (input)="query.set($any($event.target).value)" />
<input (keydown.enter)="submit()" (keydown.escape)="clear()" />   <!-- key event filtering -->
<form (submit)="$event.preventDefault(); save()">
<button (click)="select(item, $event)">Select</button>
```

`$event` is the DOM event (or the emitted value for component outputs).

### Two-way binding = property + event

```html
<app-counter [(count)]="clicks" />
<!-- desugars to -->
<app-counter [count]="clicks()" (countChange)="clicks.set($event)" />
```

With signals, use `model()` in the child (see `05-component-communication.md`).

---

# 3. Template Syntax Features

### Template expressions

Plain JS-like expressions: property access, method calls, operators, ternaries, optional chaining, `??`, pipes. **No** assignments in interpolation, no `new`, no globals like `window`/`console` (unless exposed by the component). Keep them **side-effect free and cheap** — they run during change detection.

Newer templates (v20–v22) also support: exponent `**`, `in`, `typeof`, `void`, template literals, tagged template literals, and (v22) spread/rest and inline arrow functions in some contexts — but keep logic in the class for readability.

### Template reference variables `#ref`

```html
<input #email type="email" />
<button (click)="subscribe(email.value)">Subscribe</button>

<app-video-player #player />
<button (click)="player.play()">Play</button>          <!-- reference to component instance -->

<form #f="ngForm">…</form>                             <!-- exportAs -->
```

### `@let` — local template variables (v18.1+)

```html
@let user = user$ | async;
@let fullName = user.firstName + ' ' + user.lastName;
<h2>{{ fullName }}</h2>
```

### Safe navigation & non-null assertion

```html
{{ order?.customer?.name }}        <!-- undefined-safe -->
{{ order!.id }}                    <!-- assert non-null (strictTemplates) -->
```

(v22: optional chaining now behaves like TypeScript — returns `undefined` rather than `null`.)

### Pipes in templates

```html
{{ price() | currency: "INR" : "symbol" : "1.0-0" }}
{{ createdAt() | date: "dd MMM yyyy" }}
{{ name() | uppercase }}
{{ user$ | async }}
```

See `12-pipes-and-directives.md`.

---

# 4. Lifecycle Hooks ⭐⭐⭐

Angular calls these methods at specific moments. Implement the interface for type safety.

| Order | Hook | When | Typical use |
| --- | --- | --- | --- |
| 0 | `constructor` | Class instantiated (DI happens) | `inject()` dependencies; **no** input access yet, no DOM work |
| 1 | `ngOnChanges(changes)` | Before `ngOnInit` and whenever a **decorator `@Input`** changes (by reference) | React to input changes (legacy; with signal inputs use `computed`/`effect`) |
| 2 | `ngOnInit` | Once, after first `ngOnChanges` — inputs are set | Initialization, fetching data |
| 3 | `ngDoCheck` | Every change detection run | Custom change detection (rare, expensive) |
| 4 | `ngAfterContentInit` | Once, after projected content (`ng-content`) initialized | Access `contentChild` |
| 5 | `ngAfterContentChecked` | After every check of projected content | Rare |
| 6 | `ngAfterViewInit` | Once, after the component's view & child views initialized | Access `viewChild`, DOM measurements (prefer `afterNextRender`) |
| 7 | `ngAfterViewChecked` | After every check of the view | Rare |
| 8 | `ngOnDestroy` | Before the component is destroyed | Cleanup: unsubscribe, timers, listeners |

```ts
import { Component, DestroyRef, OnDestroy, OnInit, inject } from "@angular/core";

@Component({ selector: "app-clock", template: `{{ time }}` })
export class Clock implements OnInit, OnDestroy {
  protected time = new Date().toLocaleTimeString();
  private intervalId?: ReturnType<typeof setInterval>;

  ngOnInit() {
    this.intervalId = setInterval(() => (this.time = new Date().toLocaleTimeString()), 1000);
  }

  ngOnDestroy() {
    clearInterval(this.intervalId);
  }
}
```

### `DestroyRef` — modern cleanup

```ts
export class Poller {
  constructor() {
    const id = setInterval(() => this.poll(), 5000);
    inject(DestroyRef).onDestroy(() => clearInterval(id));
  }
  private poll() {}
}
```

### Render hooks: `afterNextRender` / `afterEveryRender` ⭐

For **browser-only DOM work** after Angular renders (safe with SSR — they don't run on the server):

```ts
import { Component, ElementRef, afterNextRender, viewChild } from "@angular/core";

@Component({ selector: "app-chart", template: `<canvas #canvas></canvas>` })
export class Chart {
  private readonly canvas = viewChild.required<ElementRef<HTMLCanvasElement>>("canvas");

  constructor() {
    afterNextRender(() => {
      // runs once in the browser after the first render
      initChartLibrary(this.canvas().nativeElement);
    });
  }
}
```

- `afterNextRender(fn)` — once after the next render.
- `afterEveryRender(fn)` (formerly `afterRender`) — after every render.
- `afterRenderEffect(...)` — effect variant with read/write phases to avoid layout thrashing.

### Order with parent/child

```text
Parent constructor → Parent ngOnChanges → Parent ngOnInit → Parent ngDoCheck
  → Parent ngAfterContentInit/Checked
    → Child constructor → Child ngOnChanges → Child ngOnInit → … → Child ngAfterViewInit/Checked
  → Parent ngAfterViewInit/Checked
Destroy: children's ngOnDestroy before parent's
```

### ngOnInit vs constructor ⭐

| Constructor | `ngOnInit` |
| --- | --- |
| TypeScript class feature, called by DI | Angular lifecycle hook |
| Inputs **not** yet available | Inputs available |
| Keep it light: injection, field initializers, `effect`s | Initialization logic, data loading based on inputs |

(With signal inputs, `computed()`/`effect()` declared as fields react to inputs automatically, reducing the need for `ngOnInit`/`ngOnChanges`.)

---

# 5. View Encapsulation & Styling

| Mode | Behaviour |
| --- | --- |
| `Emulated` (default) | Styles scoped by adding generated attributes (`_ngcontent-xyz`) — no leaking in or out |
| `ShadowDom` | Uses native Shadow DOM — true isolation |
| `None` | Styles become global ⚠️ |

```css
:host { display: block; padding: 1rem; }                 /* style the host element */
:host(.featured) { border: 2px solid gold; }              /* host with class */
:host-context(.dark-theme) h3 { color: white; }           /* based on ancestor */
::ng-deep .mat-mdc-button { … }                           /* pierce encapsulation — deprecated, avoid */
```

Prefer CSS custom properties for theming child components:

```css
/* parent */ app-button { --btn-bg: rebeccapurple; }
/* child  */ button { background: var(--btn-bg, blue); }
```

---

# 6. Content Projection ⭐

Pass markup **into** a component (like React `children`/slots).

```ts
@Component({
  selector: "app-card",
  template: `
    <article class="card">
      <header><ng-content select="[card-title]" /></header>
      <section><ng-content /></section>                      <!-- default slot -->
      <footer><ng-content select="app-card-actions" /></footer>
    </article>
  `,
})
export class Card {}
```

```html
<app-card>
  <h2 card-title>Order #123</h2>
  <p>Shipped yesterday.</p>
  <app-card-actions><button>Track</button></app-card-actions>
</app-card>
```

- `select` accepts element, attribute or class selectors.
- Projected content is **created by the parent** (its lifecycle and change detection belong to the parent context) — even if not displayed (e.g. inside `@if` in the child), it's still instantiated. For conditional/lazy content, pass an `ng-template` instead (`03-control-flow-and-defer.md`).
- Fallback content (v18+): `<ng-content>Default text</ng-content>`.
- Access projected components via `contentChild()`/`contentChildren()`.

---

# 7. Host Bindings & Listeners

Prefer the `host` metadata property over `@HostBinding`/`@HostListener` decorators:

```ts
import { Component, model } from "@angular/core";

@Component({
  selector: "app-toggle",
  template: `<ng-content />`,
  host: {
    role: "switch",
    "[attr.aria-checked]": "checked()",
    "[class.on]": "checked()",
    "(click)": "toggle()",
    "(keydown.space)": "toggle(); $event.preventDefault()",
    tabindex: "0",
  },
})
export class Toggle {
  readonly checked = model(false);
  toggle() {
    this.checked.update((v) => !v);
  }
}
```

---

# 8. Dynamic Components

```ts
// Render a component type chosen at runtime
@Component({
  selector: "app-widget-host",
  imports: [NgComponentOutlet],
  template: `<ng-container *ngComponentOutlet="widget(); inputs: { data: data() }" />`,
})
export class WidgetHost {
  readonly widget = input.required<Type<unknown>>();
  readonly data = input<unknown>();
}
```

Or imperatively with `ViewContainerRef.createComponent(Comp)` — used for dialogs, toasts, plugin systems. (`createComponent` + `setInput` for inputs.)

---

# 9. Output / Behaviour Questions 🧪

**Q1.** In which hook can you first read a decorator `@Input()` value? → `ngOnChanges` (first call) / `ngOnInit`. Not in the constructor.

**Q2.** You mutate an array input (`items.push(x)`) in the parent. Does the child's `ngOnChanges` fire? → **No** — the reference didn't change. Same for OnPush children: they won't update. Create a new array.

**Q3.** Projected content inside `@if (false)` in the child — is the projected component constructed? → **Yes**, projected content is always instantiated by the parent. Use `ng-template` for lazy content.

**Q4.** `[attr.colspan]="2"` vs `[colspan]="2"`? → `colspan` has no DOM property binding equivalent named `colspan` (it's `colSpan`); use `[attr.colspan]`.

**Q5.** Order of `ngAfterViewInit` for parent and child? → Child first, then parent.

---

# 10. Interview Answers

**Explain the types of data binding in Angular.**

> "Interpolation `{{ }}` and property binding `[prop]` flow data from component to template; attribute, class and style bindings are special property bindings; event binding `(event)` flows from template to component; and two-way binding `[( )]` combines a property binding and an event binding — used with `ngModel`, or with `model()` signals between components."

**What are lifecycle hooks? Which do you use most?**

> "Methods Angular calls at specific points: `ngOnChanges` when decorator inputs change, `ngOnInit` once inputs are set, `ngDoCheck` on every check, the content and view init/checked hooks, and `ngOnDestroy` for cleanup. I mostly use `ngOnInit` for initialization and `ngOnDestroy` — or `DestroyRef` — for cleanup, and `afterNextRender` for browser-only DOM work because it's SSR-safe. With signal inputs, `computed` and `effect` replace most `ngOnChanges` logic."

**What is content projection?**

> "Passing markup from a parent into a child's template using `<ng-content>`, optionally with multiple named slots via `select`. It's how you build reusable wrappers like cards, dialogs and layouts. Projected content is created by the parent even if the child doesn't display it, so for lazy or repeated content I pass an `ng-template` instead."

**What is view encapsulation?**

> "How component styles are scoped. The default, Emulated, rewrites selectors with generated attributes so styles don't leak; ShadowDom uses native shadow roots; None makes them global. For theming children I prefer CSS custom properties over `::ng-deep`."

---

# 11. Key Takeaways

- Component = class + template + styles + metadata; standalone `imports`.
- Bindings: `{{ }}`, `[prop]`, `[attr.x]`, `[class.x]`, `[style.x]`, `(event)`, `[( )]`.
- Lifecycle: constructor → ngOnChanges → ngOnInit → … → ngAfterViewInit → ngOnDestroy; `afterNextRender` for DOM; `DestroyRef` for cleanup.
- Emulated encapsulation by default; `:host`; CSS variables for theming.
- `ng-content` slots for projection; `host` metadata instead of host decorators.

## Readiness Checklist

- [ ] Write a component with inputs, outputs, signals, host bindings
- [ ] Demonstrate every binding type
- [ ] Recite lifecycle order including parent/child
- [ ] Build a multi-slot card with `ng-content`
