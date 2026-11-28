# 03 — Control Flow, Templates & `@defer` 🔥

> **Interview + Understanding Guide** — Roadmap Section 3
>
> Goal: Use the built-in control flow blocks (`@if`, `@for`, `@switch`), deferrable views (`@defer`), and template primitives (`ng-template`, `ng-container`, `NgTemplateOutlet`) — and know the legacy structural directives they replaced.

---

# 1. Built-in Control Flow (v17+, stable) ⭐

Control flow is part of the **template syntax** — no imports needed, better type narrowing, faster and smaller than the old directives.

## 1.1 `@if`

```html
@if (user(); as u) {
  <p>Welcome, {{ u.name }}</p>
} @else if (isLoading()) {
  <app-spinner />
} @else {
  <a routerLink="/login">Log in</a>
}
```

- `as` aliases the expression result (useful with signals/async values).
- Type narrowing works inside blocks (with `strictTemplates`).

## 1.2 `@for` ⭐⭐⭐

```html
<ul>
  @for (product of products(); track product.id; let i = $index, last = $last) {
    <li [class.last]="last">{{ i + 1 }}. {{ product.name }}</li>
  } @empty {
    <li>No products found.</li>
  }
</ul>
```

- **`track` is required** — it tells Angular how to identify items so it can **reuse DOM nodes** when the list changes (like React keys).
  - `track item.id` — best (stable unique id).
  - `track $index` — only for static lists without reordering/insertion.
  - Don't track by object identity when objects are recreated (e.g. new array from the server) → every row re-renders.
- `@empty` block renders when the collection is empty.
- Contextual variables: `$index`, `$first`, `$last`, `$even`, `$odd`, `$count`.

## 1.3 `@switch`

```html
@switch (status()) {
  @case ("loading") { <app-spinner /> }
  @case ("error") { <app-error [message]="error()" /> }
  @case ("success") { <app-results [items]="items()" /> }
  @default { <p>Start searching</p> }
}
```

- Uses `===` comparison; no fall-through.
- Recent versions add exhaustiveness checking for union types (a `@default never` form flags unhandled cases at compile time) and multiple-case matching.

## 1.4 Why control flow replaced `*ngIf` / `*ngFor` / `*ngSwitch`

| Old structural directives | Built-in control flow |
| --- | --- |
| Must import `NgIf`, `NgFor`, `NgSwitch` (or `CommonModule`) | Built into the compiler |
| `trackBy` optional (often forgotten → perf bugs) | `track` **required** |
| `else` via `ng-template` references (clunky) | `@else if` / `@else` blocks |
| No `@empty` | `@empty` |
| Weaker type narrowing | Better narrowing |
| Slower diffing | Faster algorithm, smaller bundle |

`*ngIf`/`*ngFor`/`*ngSwitch` are **deprecated** (since v20). Migrate with:

```bash
ng generate @angular/core:control-flow
```

### Legacy syntax (know it for older codebases)

```html
<div *ngIf="user; else loginTpl">Hi {{ user.name }}</div>
<ng-template #loginTpl><a routerLink="/login">Log in</a></ng-template>

<li *ngFor="let p of products; trackBy: trackById; let i = index">{{ p.name }}</li>

<div [ngSwitch]="status">
  <span *ngSwitchCase="'ok'">OK</span>
  <span *ngSwitchDefault>Unknown</span>
</div>
```

The `*` is sugar: `*ngIf="cond"` desugars to `<ng-template [ngIf]="cond">…</ng-template>`.

---

# 2. Deferrable Views: `@defer` ⭐⭐⭐

Lazily load **parts of a template** (and their component/directive/pipe code) — automatic code splitting at the template level.

```html
@defer (on viewport; prefetch on idle) {
  <app-comments [postId]="postId()" />          <!-- loaded in a separate chunk -->
} @placeholder (minimum 300ms) {
  <div class="comments-placeholder">Comments</div>
} @loading (after 100ms; minimum 500ms) {
  <app-spinner />
} @error {
  <p>Failed to load comments.</p>
}
```

| Block | Shown |
| --- | --- |
| `@defer` | The deferred content (once triggered & loaded) |
| `@placeholder` | Before the trigger fires (required for some triggers like `viewport`) |
| `@loading` | While the chunk is downloading |
| `@error` | If loading fails |

`minimum` / `after` avoid flickering for fast loads.

### Triggers

| Trigger | Fires when |
| --- | --- |
| `on idle` (default) | Browser is idle (`requestIdleCallback`); v22 supports a timeout `on idle(500ms)` |
| `on viewport` | Placeholder (or a referenced element) enters the viewport |
| `on interaction` | User clicks/keys on the placeholder or a referenced element |
| `on hover` | Mouse hovers / focus |
| `on immediate` | Right after the parent renders (non-blocking) |
| `on timer(2s)` | After a delay |
| `when condition` | A boolean expression becomes true (one-time) |

```html
<button #loadBtn>Show chart</button>
@defer (on interaction(loadBtn); prefetch on hover(loadBtn)) {
  <app-heavy-chart />
}

@defer (when isAdmin()) {
  <app-admin-panel />
}
```

- **`prefetch`** downloads the code earlier than rendering (e.g. prefetch on idle, render on viewport).
- Requirements: deferred dependencies must be **standalone** and **not referenced elsewhere** in the same file outside `@defer` (otherwise they're eagerly bundled).
- **SSR**: `@defer` renders the **placeholder** on the server. With **incremental hydration** (default in v22), use `hydrate` triggers to keep server-rendered content and hydrate it lazily:

```html
@defer (hydrate on viewport) {
  <app-reviews />      <!-- server-rendered HTML; JS hydrates only when visible -->
}
```

### Use cases

- Below-the-fold sections (comments, recommendations, footers)
- Heavy widgets (charts, maps, editors) shown on interaction
- Admin/role-specific features
- Rarely used dialogs

---

# 3. `ng-template`, `ng-container`, `NgTemplateOutlet` ⭐

## 3.1 `ng-container` — grouping without a DOM element

```html
<ng-container>
  <dt>{{ term }}</dt>
  <dd>{{ definition }}</dd>
</ng-container>
```

Renders only its children (like React Fragment). Useful to apply directives without extra wrappers.

## 3.2 `ng-template` — a template fragment that isn't rendered by default

```html
<ng-template #emptyState>
  <p>No items yet.</p>
</ng-template>
```

It's rendered only when something instantiates it (a structural directive, `NgTemplateOutlet`, `ViewContainerRef`).

## 3.3 `NgTemplateOutlet` — render a template (with context)

```html
<ng-container *ngTemplateOutlet="itemTpl; context: { $implicit: item, index: i }" />

<ng-template #itemTpl let-item let-index="index">
  <span>{{ index }} — {{ item.name }}</span>
</ng-template>
```

### Pattern: customizable list (render-prop equivalent)

```ts
@Component({
  selector: "app-list",
  imports: [NgTemplateOutlet],
  template: `
    @for (item of items(); track $index) {
      <ng-container *ngTemplateOutlet="itemTemplate() ?? defaultTpl; context: { $implicit: item }" />
    }
    <ng-template #defaultTpl let-item>{{ item }}</ng-template>
  `,
})
export class List<T> {
  readonly items = input.required<T[]>();
  readonly itemTemplate = input<TemplateRef<{ $implicit: T }>>();
}
```

```html
<app-list [items]="users()" [itemTemplate]="userTpl" />
<ng-template #userTpl let-user>
  <strong>{{ user.name }}</strong> ({{ user.email }})
</ng-template>
```

Or with content projection: `contentChild(TemplateRef)` to pick up an `<ng-template>` placed between the component tags.

---

# 4. Custom Structural Directives (brief)

```ts
import { Directive, TemplateRef, ViewContainerRef, effect, inject, input } from "@angular/core";

@Directive({ selector: "[appHasRole]" })
export class HasRole {
  private readonly tpl = inject(TemplateRef);
  private readonly vcr = inject(ViewContainerRef);
  private readonly auth = inject(AuthService);
  readonly appHasRole = input.required<string>();

  constructor() {
    effect(() => {
      this.vcr.clear();
      if (this.auth.hasRole(this.appHasRole())) this.vcr.createEmbeddedView(this.tpl);
    });
  }
}
```

```html
<button *appHasRole="'admin'">Delete user</button>
```

More in `12-pipes-and-directives.md`.

---

# 5. Interview Answers

**What is the new control flow syntax and why was it introduced?**

> "Since v17 Angular has built-in `@if`, `@for` and `@switch` blocks in templates. They don't need imports, have `@else if` and `@empty`, give better type narrowing, and `@for` requires a `track` expression, which prevents a common performance bug. They're also faster and produce smaller bundles than `*ngIf`/`*ngFor`, which are now deprecated; a schematic migrates existing templates."

**Why is `track` required in `@for`?**

> "It gives each item a stable identity so Angular can match items between renders and reuse or move existing DOM nodes instead of destroying and recreating them. Tracking by a unique id is best; `$index` only works for static lists."

**What is `@defer`?**

> "Deferrable views lazy-load a section of a template along with its components' code in a separate chunk, triggered by idle time, viewport visibility, interaction, hover, a timer or a condition. It has placeholder, loading and error blocks, supports prefetching, and with SSR plus incremental hydration it can keep server-rendered HTML and hydrate it lazily. It's the easiest way to cut initial bundle size without routing changes."

**`ng-template` vs `ng-container`?**

> "`ng-container` is a grouping element that renders its children without adding a DOM node. `ng-template` defines a template fragment that isn't rendered until something instantiates it — a structural directive, `NgTemplateOutlet` or `ViewContainerRef` — which is how you pass reusable or lazily rendered markup around."

---

# 6. Key Takeaways

- Use `@if`, `@for` (with `track` + `@empty`), `@switch`; legacy `*ngX` deprecated.
- `@defer` = template-level lazy loading with triggers, prefetch, placeholder/loading/error; `hydrate` triggers for SSR.
- `ng-container` groups without DOM; `ng-template` + `NgTemplateOutlet` render templates with context.
- Custom structural directives use `TemplateRef` + `ViewContainerRef`.

## Readiness Checklist

- [ ] Convert `*ngIf/*ngFor` templates to control flow
- [ ] Explain `track` choices
- [ ] Add `@defer` with viewport trigger, prefetch, placeholder and loading blocks
- [ ] Build a list component accepting an item template
