# 23 — Angular Machine Coding 🔥🔥🔥

> **Interview + Practice Guide** — Roadmap Section 23
>
> Goal: Build common interview UIs quickly in modern Angular (standalone, signals, control flow, OnPush-by-default). Each solution is short enough to write in 30–45 minutes and highlights what interviewers look for.

**How to approach a machine coding round**

1. Clarify requirements & edge cases (empty, loading, error, keyboard, a11y) — 2 min
2. Sketch components & state (what's a signal, what's computed) — 3 min
3. Build the happy path, then edge cases
4. Talk while coding: immutability, `track`, OnPush, cleanup
5. Leave time to refactor & mention improvements

---

# 1. Todo App (CRUD + filters) ⭐

```ts
import { Component, computed, signal } from "@angular/core";

interface Todo { id: number; title: string; done: boolean }
type Filter = "all" | "active" | "done";

@Component({
  selector: "app-todos",
  template: `
    <form (submit)="add($event, input)">
      <input #input aria-label="New todo" placeholder="What needs doing?" />
      <button>Add</button>
    </form>

    <div role="group" aria-label="Filter">
      @for (f of filters; track f) {
        <button [class.active]="filter() === f" [attr.aria-pressed]="filter() === f" (click)="filter.set(f)">{{ f }}</button>
      }
    </div>

    <ul>
      @for (t of visible(); track t.id) {
        <li>
          <input type="checkbox" [checked]="t.done" (change)="toggle(t.id)" [attr.aria-label]="'Toggle ' + t.title" />
          @if (editingId() === t.id) {
            <input #edit [value]="t.title" (keydown.enter)="rename(t.id, edit.value)" (keydown.escape)="editingId.set(null)" (blur)="rename(t.id, edit.value)" />
          } @else {
            <span [class.done]="t.done" (dblclick)="editingId.set(t.id)">{{ t.title }}</span>
          }
          <button (click)="remove(t.id)" aria-label="Delete">✕</button>
        </li>
      } @empty {
        <li>No todos 🎉</li>
      }
    </ul>
    <p>{{ remaining() }} item(s) left</p>
  `,
  styles: `.done { text-decoration: line-through } .active { font-weight: bold }`,
})
export class Todos {
  protected readonly filters: Filter[] = ["all", "active", "done"];
  protected readonly todos = signal<Todo[]>([]);
  protected readonly filter = signal<Filter>("all");
  protected readonly editingId = signal<number | null>(null);

  protected readonly visible = computed(() => {
    const f = this.filter();
    return this.todos().filter((t) => (f === "all" ? true : f === "done" ? t.done : !t.done));
  });
  protected readonly remaining = computed(() => this.todos().filter((t) => !t.done).length);

  add(e: Event, input: HTMLInputElement) {
    e.preventDefault();
    const title = input.value.trim();
    if (!title) return;
    this.todos.update((list) => [...list, { id: Date.now(), title, done: false }]);
    input.value = "";
  }
  toggle(id: number) {
    this.todos.update((list) => list.map((t) => (t.id === id ? { ...t, done: !t.done } : t)));
  }
  rename(id: number, title: string) {
    if (title.trim()) this.todos.update((list) => list.map((t) => (t.id === id ? { ...t, title: title.trim() } : t)));
    this.editingId.set(null);
  }
  remove(id: number) {
    this.todos.update((list) => list.filter((t) => t.id !== id));
  }
}
```

Talking points: immutable updates, `computed` for derived lists, `track t.id`, `@empty`, keyboard + aria. Extension: persist with an `effect` to localStorage.

---

# 2. Autocomplete / Typeahead ⭐⭐⭐

```ts
@Component({
  selector: "app-autocomplete",
  imports: [ReactiveFormsModule],
  template: `
    <div class="combo">
      <input
        [formControl]="query"
        role="combobox"
        aria-autocomplete="list"
        [attr.aria-expanded]="open()"
        aria-controls="suggestions"
        [attr.aria-activedescendant]="active() >= 0 ? 'opt-' + active() : null"
        (keydown)="onKey($event)"
        (blur)="open.set(false)"
      />
      @if (loading()) { <span class="spinner" aria-live="polite">Loading…</span> }
      @if (open() && results().length) {
        <ul id="suggestions" role="listbox">
          @for (r of results(); track r.id; let i = $index) {
            <li [id]="'opt-' + i" role="option" [attr.aria-selected]="i === active()" (mousedown)="choose(r)">{{ r.name }}</li>
          }
        </ul>
      }
    </div>
  `,
})
export class Autocomplete {
  private readonly api = inject(CountryApi);
  readonly selected = output<Country>();

  protected readonly query = new FormControl("", { nonNullable: true });
  protected readonly loading = signal(false);
  protected readonly open = signal(false);
  protected readonly active = signal(-1);

  protected readonly results = toSignal(
    this.query.valueChanges.pipe(
      map((q) => q.trim()),
      debounceTime(300),
      distinctUntilChanged(),
      tap(() => { this.loading.set(true); this.active.set(-1); }),
      switchMap((q) =>
        (q.length < 2 ? of([]) : this.api.search(q)).pipe(
          catchError(() => of([])),
          finalize(() => { this.loading.set(false); this.open.set(true); }),
        ),
      ),
    ),
    { initialValue: [] as Country[] },
  );

  onKey(e: KeyboardEvent) {
    const n = this.results().length;
    if (e.key === "ArrowDown") { e.preventDefault(); this.active.update((i) => (i + 1) % n); }
    if (e.key === "ArrowUp") { e.preventDefault(); this.active.update((i) => (i - 1 + n) % n); }
    if (e.key === "Enter" && this.active() >= 0) this.choose(this.results()[this.active()]);
    if (e.key === "Escape") this.open.set(false);
  }

  choose(c: Country) {
    this.query.setValue(c.name, { emitEvent: false });
    this.open.set(false);
    this.selected.emit(c);
  }
}
```

Talking points: debounce, `switchMap` cancels stale requests (race conditions), inner `catchError`, keyboard navigation, ARIA combobox, `mousedown` before `blur`. Extensions: caching results in a `Map`, highlighting matches, `httpResource` with a debounced signal.

---

# 3. Pagination with `httpResource` ⭐⭐

```ts
interface Page<T> { items: T[]; total: number }

@Component({
  selector: "app-users-table",
  template: `
    @if (users.isLoading()) { <p>Loading…</p> }
    @if (users.error()) { <p>Error. <button (click)="users.reload()">Retry</button></p> }

    <table>
      @for (u of users.value().items; track u.id) {
        <tr><td>{{ u.name }}</td><td>{{ u.email }}</td></tr>
      }
    </table>

    <nav aria-label="Pagination">
      <button (click)="page.set(page() - 1)" [disabled]="page() === 1">Prev</button>
      @for (p of pages(); track p) {
        <button (click)="page.set(p)" [attr.aria-current]="p === page() ? 'page' : null">{{ p }}</button>
      }
      <button (click)="page.set(page() + 1)" [disabled]="page() === totalPages()">Next</button>
    </nav>
  `,
})
export class UsersTable {
  protected readonly pageSize = 10;
  protected readonly page = signal(1);

  protected readonly users = httpResource<Page<User>>(
    () => ({ url: "/api/users", params: { page: this.page(), limit: this.pageSize } }),
    { defaultValue: { items: [], total: 0 } },
  );

  protected readonly totalPages = computed(() => Math.max(1, Math.ceil(this.users.value().total / this.pageSize)));
  protected readonly pages = computed(() => Array.from({ length: this.totalPages() }, (_, i) => i + 1));
}
```

Talking points: resource refetches on `page` change and aborts stale requests; `defaultValue` avoids undefined; extension: sync `page` with query params, windowed page numbers (1 … 4 5 6 … 20).

---

# 4. Modal / Dialog (reusable, accessible) ⭐⭐

```ts
@Component({
  selector: "app-modal",
  imports: [A11yModule],                 // cdkTrapFocus
  host: { "(document:keydown.escape)": "open() && close()" },
  template: `
    @if (open()) {
      <div class="backdrop" (click)="close()"></div>
      <div class="dialog" role="dialog" aria-modal="true" [attr.aria-labelledby]="titleId" cdkTrapFocus [cdkTrapFocusAutoCapture]="true">
        <header>
          <h2 [id]="titleId"><ng-content select="[modal-title]" /></h2>
          <button (click)="close()" aria-label="Close">✕</button>
        </header>
        <ng-content />
        <footer><ng-content select="[modal-actions]" /></footer>
      </div>
    }
  `,
  styles: `
    .backdrop { position: fixed; inset: 0; background: rgb(0 0 0 / .5) }
    .dialog { position: fixed; top: 50%; left: 50%; translate: -50% -50%; background: white; padding: 1rem; border-radius: 8px; min-width: 320px }
  `,
})
export class Modal {
  readonly open = model(false);
  readonly closed = output<void>();
  protected readonly titleId = `modal-title-${Math.random().toString(36).slice(2)}`;

  close() {
    this.open.set(false);
    this.closed.emit();
  }
}
```

```html
<button (click)="showDelete.set(true)">Delete</button>
<app-modal [(open)]="showDelete">
  <span modal-title>Delete item?</span>
  <p>This cannot be undone.</p>
  <div modal-actions><button (click)="confirmDelete()">Delete</button></div>
</app-modal>
```

Talking points: `model()` for two-way open state, multi-slot projection, Escape key, focus trap + autoCapture (restores focus on close), `aria-modal`. Production: CDK `Dialog` or the native `<dialog>` element, body scroll lock, portal to `body`.

---

# 5. Tabs (content children) ⭐⭐

```ts
@Component({
  selector: "app-tab",
  template: `@if (active()) { <div role="tabpanel"><ng-content /></div> }`,
})
export class Tab {
  readonly label = input.required<string>();
  readonly active = signal(false);
}

@Component({
  selector: "app-tabs",
  template: `
    <div role="tablist" (keydown)="onKey($event)">
      @for (tab of tabs(); track tab; let i = $index) {
        <button role="tab" [attr.aria-selected]="i === selected()" [tabindex]="i === selected() ? 0 : -1" (click)="selected.set(i)">
          {{ tab.label() }}
        </button>
      }
    </div>
    <ng-content />
  `,
})
export class Tabs {
  protected readonly tabs = contentChildren(Tab);
  protected readonly selected = model(0);

  constructor() {
    effect(() => {
      const index = this.selected();
      this.tabs().forEach((t, i) => t.active.set(i === index));   // sync child state (UI side effect)
    });
  }

  onKey(e: KeyboardEvent) {
    const n = this.tabs().length;
    if (e.key === "ArrowRight") this.selected.update((i) => (i + 1) % n);
    if (e.key === "ArrowLeft") this.selected.update((i) => (i - 1 + n) % n);
  }
}
```

```html
<app-tabs>
  <app-tab label="Profile">Profile content</app-tab>
  <app-tab label="Billing">Billing content</app-tab>
</app-tabs>
```

Talking points: `contentChildren` signal query, roving tabindex, arrow-key nav, ARIA roles. Alternative: Angular Aria tabs or CDK.

---

# 6. Star Rating (custom form control) ⭐⭐

```ts
@Component({
  selector: "app-star-rating",
  template: `
    <div role="radiogroup" aria-label="Rating" (mouseleave)="hover.set(0)">
      @for (s of stars(); track s) {
        <button
          type="button"
          role="radio"
          [attr.aria-checked]="s === value()"
          [attr.aria-label]="s + ' star' + (s > 1 ? 's' : '')"
          [disabled]="disabled()"
          (mouseenter)="hover.set(s)"
          (click)="select(s)"
        >{{ s <= (hover() || value()) ? "★" : "☆" }}</button>
      }
    </div>
  `,
})
export class StarRating implements FormValueControl<number> {
  readonly value = model(0);
  readonly disabled = input(false);
  readonly touch = output<void>();
  readonly max = input(5);

  protected readonly hover = signal(0);
  protected readonly stars = computed(() => Array.from({ length: this.max() }, (_, i) => i + 1));

  select(s: number) {
    this.value.set(s === this.value() ? 0 : s);   // click again to clear
    this.touch.emit();
  }
}
// Usage: <app-star-rating [(value)]="rating" />  or with Signal Forms: <app-star-rating [formField]="f.rating" />
```

(For classic reactive forms, implement `ControlValueAccessor` — see `11-forms.md`.)

---

# 7. Countdown Timer ⭐

```ts
@Component({
  selector: "app-countdown",
  template: `
    <p aria-live="polite">{{ minutes() }}:{{ seconds() }}</p>
    <button (click)="running() ? pause() : start()">{{ running() ? "Pause" : "Start" }}</button>
    <button (click)="reset()">Reset</button>
  `,
})
export class Countdown {
  readonly from = input(60);                     // seconds
  readonly finished = output<void>();

  private readonly remaining = linkedSignal(() => this.from());   // resets if input changes
  protected readonly running = signal(false);
  protected readonly minutes = computed(() => String(Math.floor(this.remaining() / 60)).padStart(2, "0"));
  protected readonly seconds = computed(() => String(this.remaining() % 60).padStart(2, "0"));

  private intervalId: ReturnType<typeof setInterval> | undefined;

  constructor() {
    inject(DestroyRef).onDestroy(() => this.pause());
  }

  start() {
    if (this.running() || this.remaining() === 0) return;
    this.running.set(true);
    this.intervalId = setInterval(() => {
      this.remaining.update((r) => r - 1);
      if (this.remaining() <= 0) {
        this.pause();
        this.finished.emit();
      }
    }, 1000);
  }

  pause() {
    clearInterval(this.intervalId);
    this.running.set(false);
  }

  reset() {
    this.pause();
    this.remaining.set(this.from());
  }
}
```

Talking points: `linkedSignal` for resettable state, signals make it zoneless-safe, cleanup with `DestroyRef`. Accuracy improvement: compute from `Date.now()` deltas instead of counting ticks (timers drift, background tabs throttle).

---

# 8. File Explorer / Nested Tree (recursive component) ⭐⭐

```ts
export interface TreeNode { name: string; children?: TreeNode[] }

@Component({
  selector: "app-tree-node",
  imports: [forwardRef(() => TreeNodeView)],
  template: `
    <div class="row" [style.padding-left.px]="depth() * 16">
      @if (node().children?.length) {
        <button (click)="expanded.set(!expanded())" [attr.aria-expanded]="expanded()">{{ expanded() ? "▾" : "▸" }}</button>📁
      } @else {
        <span class="spacer"></span>📄
      }
      {{ node().name }}
    </div>
    @if (expanded()) {
      @for (child of node().children ?? []; track child.name) {
        <app-tree-node [node]="child" [depth]="depth() + 1" />
      }
    }
  `,
})
export class TreeNodeView {
  readonly node = input.required<TreeNode>();
  readonly depth = input(0);
  protected readonly expanded = signal(false);
}
```

Talking points: recursion with self-import, depth-based indentation, lazily rendering collapsed children. Extensions: `role="tree"`/`treeitem` + keyboard (or Angular Aria Tree / CDK Tree), add/rename/delete via a store, search with auto-expand.

---

# 9. Registration Form with Signal Forms ⭐⭐

```ts
interface RegistrationModel { name: string; email: string; password: string; confirm: string; terms: boolean }

@Component({
  selector: "app-registration",
  imports: [FormField],
  template: `
    <form (submit)="save($event)" novalidate>
      <label>Name <input [formField]="f.name" /></label>
      <app-field-errors [field]="f.name" />

      <label>Email <input type="email" [formField]="f.email" /></label>
      <app-field-errors [field]="f.email" />

      <label>Password <input type="password" [formField]="f.password" /></label>
      <label>Confirm <input type="password" [formField]="f.confirm" /></label>
      <app-field-errors [field]="f.confirm" />

      <label><input type="checkbox" [formField]="f.terms" /> I accept the terms</label>

      <button [disabled]="f().invalid() || f().submitting()">{{ f().submitting() ? "Saving…" : "Register" }}</button>
    </form>
  `,
})
export class RegistrationForm {
  private readonly api = inject(AuthApi);
  protected readonly model = signal<RegistrationModel>({ name: "", email: "", password: "", confirm: "", terms: false });

  protected readonly f = form(this.model, (p) => {
    required(p.name);
    required(p.email);
    email(p.email);
    minLength(p.password, 8);
    pattern(p.password, /\d/, { message: "Include a number" });
    validate(p.confirm, ({ value, valueOf }) => (value() === valueOf(p.password) ? undefined : { kind: "mismatch", message: "Passwords must match" }));
    validate(p.terms, ({ value }) => (value() ? undefined : { kind: "terms", message: "Required" }));
  });

  protected async save(e: Event) {
    e.preventDefault();
    await submit(this.f, async () => {
      await firstValueFrom(this.api.register(this.model()));
      return undefined;
    });
  }
}
```

Talking points: model signal as source of truth, schema-based validation incl. cross-field, `submitting()` state, reusable error component.

---

# 10. Infinite Scroll ⭐⭐

```ts
@Component({
  selector: "app-feed",
  template: `
    @for (post of posts(); track post.id) { <app-post-card [post]="post" /> }
    @if (loading()) { <p>Loading…</p> }
    @if (!hasMore()) { <p>You're all caught up</p> }
    <div #sentinel aria-hidden="true"></div>
  `,
})
export class Feed {
  private readonly api = inject(PostApi);
  private readonly sentinel = viewChild.required<ElementRef<HTMLElement>>("sentinel");

  protected readonly posts = signal<Post[]>([]);
  protected readonly loading = signal(false);
  protected readonly hasMore = signal(true);
  private readonly destroyRef = inject(DestroyRef);   // captured in injection context
  private page = 0;

  constructor() {
    afterNextRender(() => {
      const observer = new IntersectionObserver(([entry]) => entry.isIntersecting && this.loadMore(), { rootMargin: "200px" });
      observer.observe(this.sentinel().nativeElement);
      this.destroyRef.onDestroy(() => observer.disconnect());   // ❗ calling inject() here would throw NG0203
    });
  }

  loadMore() {
    if (this.loading() || !this.hasMore()) return;
    this.loading.set(true);
    this.api.list(++this.page).subscribe({
      next: (items) => {
        this.posts.update((p) => [...p, ...items]);
        this.hasMore.set(items.length > 0);
        this.loading.set(false);
      },
      error: () => {
        this.page--;
        this.loading.set(false);
      },
    });
  }
}
```

Talking points: `IntersectionObserver` over scroll events, `afterNextRender` (SSR-safe), guard against concurrent loads, error rollback, cleanup. Extension: CDK virtual scroll for huge lists.

---

# More Practice Ideas

- Kanban board with CDK drag-drop + signal store
- Shopping cart with signal store & persisted state
- Data table with sorting, filtering, pagination (computed pipeline)
- Debounced search with `httpResource` + `debounced()`
- Stepper/wizard with component-scoped state service
- Toast notification service with dynamic components / overlay
- Accordion with `contentChildren` and keyboard support
- Polling dashboard with `timer` + `switchMap`
- Dark-mode toggle persisted with `effect` + SSR-safe storage
