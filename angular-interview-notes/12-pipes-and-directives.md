# 12 — Pipes & Directives 🔥

> **Interview + Understanding Guide** — Roadmap Section 12
>
> Goal: Use and build pipes (pure vs impure, async, custom) and directives (attribute, structural, host directives / composition), and know when each beats a component or method call.

---

# 1. Pipes ⭐

A **pipe** transforms a value for display in a template: `{{ value | pipeName:arg1:arg2 }}`.

## 1.1 Built-in pipes

```html
{{ title | uppercase }} {{ title | lowercase }} {{ title | titlecase }}
{{ price | currency:'EUR':'symbol':'1.2-2' }}      <!-- €1,234.50 -->
{{ ratio | percent:'1.0-1' }}
{{ 3.14159 | number:'1.2-2' }}                     <!-- minInt.minFrac-maxFrac -->
{{ createdAt | date:'mediumDate' }} {{ createdAt | date:'dd/MM/yyyy HH:mm' }}
{{ user | json }}                                   <!-- debugging -->
{{ items | slice:0:5 }}
{{ obj | keyvalue }}
{{ data$ | async }}
{{ count | i18nPlural: { '=0': 'No items', '=1': 'One item', 'other': '# items' } }}
```

Import from `@angular/common` in standalone components (`imports: [DatePipe, CurrencyPipe, AsyncPipe]`).

## 1.2 `async` pipe ⭐

```html
@if (user$ | async; as user) {
  <h2>{{ user.name }}</h2>
}
```

- Subscribes to an Observable/Promise, returns latest value
- **Unsubscribes automatically** on destroy or when the source changes
- Calls `markForCheck()` on each emission → works with OnPush and zoneless
- ⚠️ Each `| async` on the same cold observable = separate subscription (duplicate HTTP)

## 1.3 Custom pipe

```ts
import { Pipe, PipeTransform } from "@angular/core";

@Pipe({ name: "truncate" })            // standalone by default
export class TruncatePipe implements PipeTransform {
  transform(value: string | null | undefined, limit = 50, suffix = "…"): string {
    if (!value) return "";
    return value.length > limit ? value.slice(0, limit).trimEnd() + suffix : value;
  }
}
```

```html
<p>{{ post.body | truncate:100 }}</p>
```

```ts
@Pipe({ name: "timeAgo" })
export class TimeAgoPipe implements PipeTransform {
  private readonly rtf = new Intl.RelativeTimeFormat("en", { numeric: "auto" });
  transform(date: Date | string): string {
    const diffSec = Math.round((new Date(date).getTime() - Date.now()) / 1000);
    const units: [Intl.RelativeTimeFormatUnit, number][] = [["year", 31536000], ["month", 2592000], ["day", 86400], ["hour", 3600], ["minute", 60]];
    for (const [unit, sec] of units) if (Math.abs(diffSec) >= sec) return this.rtf.format(Math.round(diffSec / sec), unit);
    return this.rtf.format(diffSec, "second");
  }
}
```

## 1.4 Pure vs Impure ⭐⭐⭐

| | Pure (default) | Impure (`pure: false`) |
| --- | --- | --- |
| Re-runs when | Input **reference/primitive value** changes | **Every** change detection cycle |
| Caching | Memoized per input | None |
| Mutations detected | ❌ (`array.push` not seen) | ✅ |
| Cost | Cheap | Potentially expensive |
| Example | `date`, `currency`, custom formatters | `async`, `json`, `keyvalue`, `slice` on mutable data |

```ts
@Pipe({ name: "filterBy" })    // pure
export class FilterByPipe implements PipeTransform {
  transform<T>(items: T[], predicate: (t: T) => boolean) { return items.filter(predicate); }
}
// items.push(x) → pipe doesn't re-run (same array reference). Use immutable updates or a computed signal.
```

### Pipe vs method call in template ⭐

```html
{{ formatPrice(product.price) }}   <!-- ❌ method runs on EVERY change detection -->
{{ product.price | price }}         <!-- ✅ pure pipe runs only when price changes -->
{{ formattedPrice() }}               <!-- ✅ computed signal — also memoized -->
```

With signals, `computed` often replaces custom pipes for component-specific derivations; pipes remain great for reusable formatting.

---

# 2. Directives ⭐

Directives add behaviour to elements. Three kinds:

| Kind | Example | What it does |
| --- | --- | --- |
| **Components** | `<app-card>` | Directive with a template |
| **Attribute directives** | `ngClass`, `ngStyle`, `ngModel`, custom `appHighlight` | Change appearance/behaviour of an element |
| **Structural directives** | `*ngIf`, `*ngFor` (legacy), custom `*appRole` | Add/remove DOM via `ng-template` |

## 2.1 Built-in attribute directives

```html
<div [ngClass]="{ active: isActive(), disabled: isDisabled() }"></div>
<div [class.active]="isActive()"></div>          <!-- preferred for single classes -->
<div [class]="{ active: isActive() }"></div>     <!-- class binding accepts objects too -->
<div [ngStyle]="{ 'width.px': width() }"></div>
<div [style.width.px]="width()"></div>           <!-- preferred -->
```

Modern guidance: prefer `[class]` / `[style]` bindings over `NgClass`/`NgStyle`.

## 2.2 Custom attribute directive

```ts
import { Directive, ElementRef, inject, input } from "@angular/core";

@Directive({
  selector: "[appHighlight]",
  host: {
    "(mouseenter)": "setColor(color())",
    "(mouseleave)": "setColor('')",
  },
})
export class Highlight {
  readonly color = input("yellow", { alias: "appHighlight" });
  private readonly el = inject<ElementRef<HTMLElement>>(ElementRef);

  setColor(c: string) {
    this.el.nativeElement.style.backgroundColor = c;
  }
}
```

```html
<p appHighlight="lightblue">Hover me</p>
```

Prefer `host` metadata over `@HostListener`/`@HostBinding` decorators (still supported). Use `Renderer2` for DOM changes that must work across platforms (SSR) — or better, host bindings:

```ts
@Directive({
  selector: "[appAutoFocus]",
})
export class AutoFocus {
  private readonly el = inject<ElementRef<HTMLInputElement>>(ElementRef);
  constructor() {
    afterNextRender(() => this.el.nativeElement.focus());   // browser-only, after render
  }
}
```

### Click-outside directive (frequently asked)

```ts
@Directive({
  selector: "[appClickOutside]",
  host: { "(document:click)": "onDocumentClick($event)" },
})
export class ClickOutside {
  readonly appClickOutside = output<void>();
  private readonly el = inject(ElementRef);

  onDocumentClick(event: MouseEvent) {
    if (!this.el.nativeElement.contains(event.target)) this.appClickOutside.emit();
  }
}
// <div class="dropdown" (appClickOutside)="close()">…</div>
```

## 2.3 Custom structural directive

```ts
@Directive({ selector: "[appIfRole]" })
export class IfRole {
  private readonly tpl = inject(TemplateRef);
  private readonly vcr = inject(ViewContainerRef);
  private readonly auth = inject(AuthStore);
  readonly appIfRole = input.required<string>();

  constructor() {
    effect(() => {
      this.vcr.clear();
      if (this.auth.roles().includes(this.appIfRole())) this.vcr.createEmbeddedView(this.tpl);
    });
  }
}
```

```html
<button *appIfRole="'admin'">Delete</button>
<!-- desugars to: <ng-template [appIfRole]="'admin'"><button>Delete</button></ng-template> -->
```

Typed template context: implement `static ngTemplateContextGuard(dir, ctx): ctx is MyContext`.

---

# 3. Host Directives (Directive Composition API) ⭐

Compose reusable behaviours onto a component **without inheritance**:

```ts
@Directive({ selector: "[appTooltip]", host: { "(mouseenter)": "show()", "(mouseleave)": "hide()" } })
export class Tooltip {
  readonly tooltipText = input("");
  show() {}
  hide() {}
}

@Directive({ selector: "[appFocusRing]", host: { "[class.focus-ring]": "true" } })
export class FocusRing {}

@Component({
  selector: "app-icon-button",
  template: `<ng-content />`,
  hostDirectives: [
    FocusRing,
    { directive: Tooltip, inputs: ["tooltipText: tooltip"] },   // expose input under alias
  ],
})
export class IconButton {}
// <app-icon-button tooltip="Save">💾</app-icon-button>
```

- Host directives must be **standalone**; inputs/outputs aren't exposed unless listed.
- Used heavily by Angular CDK/Material/Aria for behaviours (focus, keyboard nav, ARIA).

---

# 4. Directive vs Component vs Pipe

| Use | When |
| --- | --- |
| **Component** | You need a template/UI |
| **Attribute directive** | Add behaviour/styling to existing elements (focus, tooltip, permissions, click-outside) |
| **Structural directive** | Conditionally/repeatedly render a template |
| **Pipe** | Pure display transformation of a value |
| **`computed`** | Component-specific derived value |

---

# 5. Interview Answers

**Pure vs impure pipes?**

> "A pure pipe — the default — only re-runs when its input changes by reference or primitive value, so Angular memoizes it; mutating an array won't trigger it. An impure pipe runs on every change detection cycle, which detects mutations but can be expensive. `async` is impure because it must check for new emissions. I keep custom pipes pure and use immutable data."

**Why use a pipe instead of a method in the template?**

> "A method call in a template runs on every change detection cycle. A pure pipe only recomputes when its input changes, so it's cheaper and reusable across components. For component-specific derivations a `computed` signal gives the same memoization."

**Types of directives? How do you create a structural directive?**

> "Components, attribute directives and structural directives. A structural directive injects `TemplateRef` and `ViewContainerRef` and creates or clears embedded views based on its input — the `*` syntax is sugar for wrapping the element in an `ng-template`. Since v17 the built-in control flow covers most cases, but custom ones are useful for things like permission-based rendering."

**What are host directives?**

> "The directive composition API: a component lists standalone directives in `hostDirectives` and they're applied to its host element, optionally exposing their inputs and outputs. It's composition instead of inheritance for reusable behaviours like tooltips, focus handling or ARIA patterns."

---

# 6. Key Takeaways

- Pure pipes are memoized; impure run every cycle; `async` handles subscriptions + markForCheck.
- Pipes/computed over template method calls.
- Attribute directives (`host` metadata, `inject(ElementRef)`), structural (`TemplateRef` + `ViewContainerRef`).
- `hostDirectives` = composition.

## Readiness Checklist

- [ ] Build `truncate`, `timeAgo` pipes
- [ ] Demonstrate the pure pipe + mutation gotcha
- [ ] Build highlight, click-outside and `*appIfRole` directives
- [ ] Compose behaviours with `hostDirectives`
