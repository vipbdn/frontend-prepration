# 17 — Angular Material, CDK & Aria 🔥

> **Interview + Understanding Guide** — Roadmap Section 17
>
> Goal: Know the Angular component ecosystem — Material (styled components, theming), CDK (behaviour primitives: overlay, drag-drop, virtual scroll, a11y) and Angular Aria (headless accessible components, GA in v22) — and accessibility fundamentals.

---

# 1. The Three Layers ⭐

| Package | What | Styling |
| --- | --- | --- |
| **Angular Material** (`@angular/material`) | Ready-made Material Design 3 components (button, form field, dialog, table, datepicker…) | Opinionated, themable via design tokens |
| **CDK** (`@angular/cdk`) | Behaviour primitives used to build components (overlay, portal, drag-drop, virtual scroll, a11y, layout, table) | None — bring your own |
| **Angular Aria** (`@angular/aria`) 🆕 GA in v22 | **Headless**, accessible UI patterns (combobox, listbox, menu, tabs, tree, grid, accordion, toolbar…) implementing WAI-ARIA keyboard/focus behaviour | None — fully custom design |

Rule: need Material look → Material; custom design system → Aria/CDK.

---

# 2. Angular Material

```bash
ng add @angular/material
```

```ts
@Component({
  imports: [MatButtonModule, MatFormFieldModule, MatInputModule, MatIconModule, ReactiveFormsModule],
  template: `
    <mat-form-field appearance="outline">
      <mat-label>Email</mat-label>
      <input matInput [formControl]="email" type="email" />
      @if (email.hasError("email")) { <mat-error>Invalid email</mat-error> }
    </mat-form-field>
    <button mat-flat-button (click)="save()"><mat-icon>save</mat-icon> Save</button>
  `,
})
export class Settings {
  email = new FormControl("");
  save() {}
}
```

### Dialog

```ts
const ref = inject(MatDialog).open(ConfirmDialog, { data: { message: "Delete?" }, width: "400px" });
ref.afterClosed().subscribe((confirmed) => confirmed && this.delete());

// inside ConfirmDialog
readonly data = inject<{ message: string }>(MAT_DIALOG_DATA);
readonly ref = inject(MatDialogRef<ConfirmDialog, boolean>);
```

### Table with sort & paginator

```html
<table mat-table [dataSource]="dataSource" matSort>
  <ng-container matColumnDef="name">
    <th mat-header-cell *matHeaderCellDef mat-sort-header>Name</th>
    <td mat-cell *matCellDef="let row">{{ row.name }}</td>
  </ng-container>
  <tr mat-header-row *matHeaderRowDef="columns"></tr>
  <tr mat-row *matRowDef="let row; columns: columns"></tr>
</table>
<mat-paginator [pageSize]="10" />
```

### Theming (Material 3)

```scss
@use "@angular/material" as mat;

html {
  color-scheme: light dark;
  @include mat.theme((color: mat.$violet-palette, typography: Roboto, density: 0));
}

// override tokens per component
.danger-button {
  @include mat.button-overrides((filled-container-color: #b3261e));
}
```

Theming uses **CSS custom properties (system tokens)**, so you can customize with plain CSS variables (`--mat-sys-primary`) and support dark mode with `color-scheme`.

---

# 3. CDK — Component Dev Kit ⭐

| Module | Use |
| --- | --- |
| **Overlay** | Floating panels positioned relative to elements (dropdowns, tooltips, popovers) |
| **Portal** | Render content elsewhere in the DOM |
| **Dialog** (`@angular/cdk/dialog`) | Unstyled accessible dialogs |
| **Drag & Drop** | Sortable lists, kanban boards |
| **Scrolling** | Virtual scroll (`cdk-virtual-scroll-viewport`) |
| **A11y** | `FocusTrap`, `LiveAnnouncer`, `FocusMonitor`, `ListKeyManager` |
| **Layout** | `BreakpointObserver` for responsive logic |
| **Table** | Unstyled data table foundation |
| **Clipboard**, **Stepper**, **Menu**, **Listbox**, **Testing (harnesses)** | … |

### Drag & drop example

```ts
import { CdkDrag, CdkDropList, CdkDragDrop, moveItemInArray } from "@angular/cdk/drag-drop";

@Component({
  imports: [CdkDropList, CdkDrag],
  template: `
    <ul cdkDropList (cdkDropListDropped)="drop($event)">
      @for (task of tasks(); track task) { <li cdkDrag>{{ task }}</li> }
    </ul>
  `,
})
export class TaskList {
  readonly tasks = signal(["Design", "Build", "Test"]);
  drop(e: CdkDragDrop<string[]>) {
    this.tasks.update((list) => {
      const copy = [...list];
      moveItemInArray(copy, e.previousIndex, e.currentIndex);
      return copy;
    });
  }
}
```

### Responsive with BreakpointObserver

```ts
readonly isHandset = toSignal(
  inject(BreakpointObserver).observe(Breakpoints.Handset).pipe(map((r) => r.matches)),
  { initialValue: false },
);
```

### Overlay example (simple popover)

```ts
const overlayRef = inject(Overlay).create({
  positionStrategy: inject(Overlay).position().flexibleConnectedTo(triggerEl).withPositions([
    { originX: "start", originY: "bottom", overlayX: "start", overlayY: "top" },
  ]),
  hasBackdrop: true,
  scrollStrategy: inject(Overlay).scrollStrategies.reposition(),
});
overlayRef.attach(new ComponentPortal(MenuPanel));
overlayRef.backdropClick().subscribe(() => overlayRef.detach());
```

---

# 4. Angular Aria (headless, GA in v22) 🆕

Directives that add correct **roles, ARIA attributes, keyboard navigation and focus management** — you provide all markup and CSS.

```html
<!-- tabs pattern (illustrative) -->
<div ngTabs>
  <div ngTabList selectionMode="follow">
    <button ngTab value="profile">Profile</button>
    <button ngTab value="billing">Billing</button>
  </div>
  <div ngTabPanel value="profile">…</div>
  <div ngTabPanel value="billing">…</div>
</div>
```

Patterns: Accordion, Combobox/Autocomplete, Grid, Listbox, Menu/Menubar, Multiselect, Select, Tabs, Toolbar, Tree. Ideal for teams with their own design system who need robust accessibility.

---

# 5. Accessibility (a11y) Essentials ⭐

- Semantic HTML first (`<button>`, `<nav>`, `<main>`, `<label for>`) — not clickable `<div>`s
- Every input has a label; errors linked via `aria-describedby`; `aria-invalid`
- Keyboard: everything reachable via Tab, visible focus styles, logical order, Escape closes dialogs
- Focus management: trap focus in modals (`cdkTrapFocus`), return focus on close, move focus on route change
- Announce dynamic changes: `LiveAnnouncer` / `aria-live`
- Color contrast (WCAG AA 4.5:1), don't rely on color alone
- `alt` text for images; `aria-hidden` for decorative icons
- Angular helps: `[attr.aria-*]` bindings, Material/Aria components are accessible, `ariaCurrentWhenActive` on router links, compiler a11y checks/ESLint `@angular-eslint/template/accessibility-*`
- Test with axe, Lighthouse, screen readers (NVDA/VoiceOver)

```html
<button [attr.aria-expanded]="open()" aria-controls="menu" (click)="open.set(!open())">Menu</button>
<ul id="menu" [hidden]="!open()" role="menu">…</ul>
```

---

# 6. Interview Answers

**Material vs CDK vs Aria?**

> "Material is a set of styled Material Design components you theme via design tokens. The CDK is the unstyled behaviour layer underneath — overlays, portals, drag and drop, virtual scrolling, focus trapping and other a11y utilities — for building your own components. Angular Aria, GA in v22, offers headless accessible patterns like combobox, menu, tabs and tree with correct ARIA and keyboard handling, so a custom design system can be fully accessible without Material's look."

**How do you make an Angular app accessible?**

> "Start with semantic HTML and labelled form controls, ensure full keyboard support with visible focus, manage focus in dialogs and on navigation, announce dynamic updates with live regions or LiveAnnouncer, maintain colour contrast, and use accessible building blocks — Material, CDK a11y or Angular Aria. I verify with angular-eslint accessibility rules, axe and screen reader testing."

---

# 7. Key Takeaways

- Material = styled; CDK = behaviour primitives; Aria = headless accessible patterns (GA v22).
- Material 3 theming via `mat.theme` and CSS tokens.
- CDK: overlay, drag-drop, virtual scroll, a11y, layout.
- a11y: semantics, labels, keyboard, focus, live regions, contrast.

## Readiness Checklist

- [ ] Build a Material form + dialog + table
- [ ] Build a drag-and-drop list with CDK
- [ ] Explain when you'd choose Aria over Material
- [ ] Audit a page with axe
