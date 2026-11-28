# 20 — NgModules & Legacy Angular 🔥

> **Interview + Understanding Guide** — Roadmap Section 20
>
> Goal: Many jobs maintain older Angular code. Understand NgModules, `forRoot`/`forChild`, decorator-based APIs, constructor DI, `*ngIf`/`*ngFor`, class guards/interceptors — and how to **migrate** to modern Angular.

---

# 1. NgModules ⭐⭐

Before standalone (v14 preview, v15 stable, default v19), every component had to belong to an **NgModule**.

```ts
@NgModule({
  declarations: [AppComponent, ProductListComponent, HighlightDirective, TruncatePipe], // components/directives/pipes owned by this module
  imports: [BrowserModule, HttpClientModule, FormsModule, AppRoutingModule, SharedModule], // other modules whose exports we use
  providers: [AuthService, { provide: HTTP_INTERCEPTORS, useClass: AuthInterceptor, multi: true }],
  exports: [],                       // what importing modules can use
  bootstrap: [AppComponent],         // root component (root module only)
})
export class AppModule {}

// main.ts
platformBrowserDynamic().bootstrapModule(AppModule);
```

| Property | Meaning |
| --- | --- |
| `declarations` | Components/directives/pipes that **belong** to this module (each declared in exactly one module) |
| `imports` | Modules whose **exported** declarables are available to this module's templates |
| `exports` | Declarables/modules made available to importers |
| `providers` | Services registered in the module's injector (root for eagerly loaded modules) |
| `bootstrap` | Root component |

### Typical module types

| Module | Purpose |
| --- | --- |
| `AppModule` | Root, bootstraps app |
| `CoreModule` | Singletons (imported once in AppModule, guarded against re-import) |
| `SharedModule` | Re-exports common components/pipes/`CommonModule`/`FormsModule` (no providers!) |
| Feature modules | Per domain, often lazy-loaded |
| Routing modules | `RouterModule.forRoot/forChild` |

### `forRoot()` / `forChild()` ⭐

```ts
@NgModule({ imports: [RouterModule.forRoot(routes)], exports: [RouterModule] })
export class AppRoutingModule {}

@NgModule({ imports: [RouterModule.forChild(adminRoutes)], exports: [RouterModule] })
export class AdminRoutingModule {}
```

**Why:** `forRoot` returns `ModuleWithProviders` including **singleton providers** (e.g. `Router`); `forChild` only registers routes/declarables. Importing providers in lazy modules would create **duplicate instances** in the lazy module's child injector.

```ts
@NgModule({})
export class ToastModule {
  static forRoot(config: ToastConfig): ModuleWithProviders<ToastModule> {
    return { ngModule: ToastModule, providers: [{ provide: TOAST_CONFIG, useValue: config }, ToastService] };
  }
}
```

### Lazy module (legacy)

```ts
{ path: "admin", loadChildren: () => import("./admin/admin.module").then((m) => m.AdminModule) }
```

### Problems NgModules caused (why standalone)

- Extra boilerplate and indirection ("which module declares this?")
- Hard to know a component's real dependencies
- Big SharedModules hurt tree-shaking
- Steeper learning curve

---

# 2. Standalone vs NgModule (comparison)

```ts
// Standalone (modern)
@Component({
  selector: "app-product-list",
  imports: [ProductCard, CurrencyPipe, RouterLink],   // direct dependencies
  template: `…`,
})
export class ProductList {}

bootstrapApplication(App, appConfig);
```

Interop: standalone components can be **imported into NgModules**, and NgModules can be **imported into standalone components**' `imports`. `importProvidersFrom(SomeModule.forRoot())` brings module providers into `bootstrapApplication`.

---

# 3. Legacy APIs → Modern Equivalents ⭐⭐⭐

| Legacy | Modern |
| --- | --- |
| `@NgModule` declarations | `standalone` components (default since v19) |
| `platformBrowserDynamic().bootstrapModule(AppModule)` | `bootstrapApplication(App, appConfig)` |
| `HttpClientModule` | `provideHttpClient()` |
| `RouterModule.forRoot(routes)` | `provideRouter(routes)` |
| `BrowserAnimationsModule` | `provideAnimationsAsync()` → (v20.2+) native CSS `animate.enter`/`animate.leave` |
| `@Input()` / `@Output()` + `EventEmitter` | `input()` / `output()` / `model()` |
| `ngOnChanges` for derived values | `computed()` |
| `@ViewChild` / `@ContentChildren` / `QueryList` | `viewChild()` / `contentChildren()` |
| `@HostBinding` / `@HostListener` | `host: {}` metadata |
| Constructor DI | `inject()` |
| `@Injectable({ providedIn: "root" })` | `@Service()` (v22) |
| `*ngIf` / `*ngFor` / `*ngSwitch` (deprecated v20) | `@if` / `@for` / `@switch` |
| Class guards (`implements CanActivate`) | Functional guards |
| Class interceptors + `HTTP_INTERCEPTORS` | Functional interceptors + `withInterceptors` |
| `APP_INITIALIZER` | `provideAppInitializer()` |
| `BehaviorSubject` service state | Signal stores |
| `async` pipe everywhere | `toSignal`, resources |
| Default change detection + zone.js | OnPush (default v22) + zoneless (default v21) |
| Karma/Jasmine | Vitest (default v21) |
| Webpack builder (`browser`) | esbuild `application` builder (webpack deprecated v22) |
| `ChangeDetectionStrategy.Default` | `ChangeDetectionStrategy.Eager` (v22 rename) |

---

# 4. Legacy Code You Must Read

```ts
@Component({
  selector: "app-user-list",
  templateUrl: "./user-list.component.html",
})
export class UserListComponent implements OnInit, OnChanges, OnDestroy {
  @Input() users: User[] = [];
  @Input() filter = "";
  @Output() selected = new EventEmitter<User>();
  @ViewChild("search") searchInput!: ElementRef<HTMLInputElement>;
  @HostBinding("class.compact") compact = false;

  filtered: User[] = [];
  private destroy$ = new Subject<void>();

  constructor(private userService: UserService, private cdr: ChangeDetectorRef) {}

  ngOnChanges(changes: SimpleChanges) {
    if (changes["users"] || changes["filter"]) {
      this.filtered = this.users.filter((u) => u.name.includes(this.filter));
    }
  }

  ngOnInit() {
    this.userService.refresh$.pipe(takeUntil(this.destroy$)).subscribe(() => this.cdr.markForCheck());
  }

  @HostListener("window:resize")
  onResize() {
    this.compact = window.innerWidth < 600;
  }

  ngOnDestroy() {
    this.destroy$.next();
    this.destroy$.complete();
  }
}
```

```html
<li *ngFor="let user of filtered; trackBy: trackById; let i = index">{{ i + 1 }}. {{ user.name }}</li>
<p *ngIf="loading; else content">Loading…</p>
<ng-template #content>…</ng-template>
<div [ngSwitch]="status"><span *ngSwitchCase="'ok'">OK</span><span *ngSwitchDefault>?</span></div>
```

### Modernized version

```ts
@Component({
  selector: "app-user-list",
  templateUrl: "./user-list.html",
  host: { "[class.compact]": "compact()", "(window:resize)": "updateCompact()" },
})
export class UserList {
  readonly users = input<User[]>([]);
  readonly filter = input("");
  readonly selected = output<User>();
  private readonly searchInput = viewChild<ElementRef<HTMLInputElement>>("search");

  protected readonly filtered = computed(() => this.users().filter((u) => u.name.includes(this.filter())));
  protected readonly compact = signal(false);

  protected updateCompact() {
    this.compact.set(window.innerWidth < 600);
  }
}
```

---

# 5. Migration Strategy ⭐⭐

Angular ships schematics — run them incrementally, one at a time, with tests between:

```bash
ng update @angular/core @angular/cli                      # one major at a time
ng generate @angular/core:standalone                       # NgModules → standalone (3 steps)
ng generate @angular/core:control-flow                     # *ngIf/*ngFor → @if/@for
ng generate @angular/core:signal-input-migration           # @Input → input()
ng generate @angular/core:output-migration                 # @Output → output()
ng generate @angular/core:signal-queries-migration         # @ViewChild → viewChild()
ng generate @angular/core:inject                           # constructor → inject()
ng generate @angular/core:route-lazy-loading               # eager routes → lazy
ng generate @angular/core:self-closing-tag                 # <app-x></app-x> → <app-x />
ng generate @angular/core:cleanup-unused-imports
```

Plan for a large legacy app:
1. Update Angular version step by step (`update.angular.dev` guide), fix deprecations.
2. Switch to the `application` (esbuild) builder.
3. Run standalone migration; replace `HttpClientModule`/`RouterModule` with providers.
4. Control flow migration.
5. Signal inputs/outputs/queries, `inject()`.
6. New code: signals, OnPush, functional APIs; gradually move state to signals.
7. Make components zoneless-compatible, then enable zoneless.
8. Migrate tests to Vitest.

Interop helpers keep old and new code coexisting (`toSignal`, `importProvidersFrom`, standalone in NgModules).

---

# 6. Interview Answers

**What is an NgModule? Is it still needed?**

> "An NgModule groups declarables — components, directives, pipes — with `declarations`, `imports`, `exports`, `providers` and `bootstrap`, and defines a compilation context. Since v15 standalone components declare their own dependencies via `imports`, and since v19 they're the default, so new apps don't need NgModules. They're still supported and common in existing codebases, and the two interoperate."

**Why `forRoot` and `forChild`?**

> "To avoid duplicate singleton providers. `forRoot` returns the module together with its providers and is imported once in the root; `forChild` returns only declarables and routes for lazy feature modules. If a lazy module imported the providers, its child injector would create a second instance."

**How would you migrate a legacy Angular app?**

> "Incrementally: update one major version at a time with `ng update`, switch to the esbuild application builder, then run Angular's schematics — standalone, control flow, signal inputs/outputs/queries, `inject()`, lazy routes — testing after each. New code uses signals, OnPush and functional APIs, and once state flows through signals I enable zoneless. Interop APIs let old and new coexist so there's no big-bang rewrite."

---

# 7. Key Takeaways

- NgModule: declarations / imports / exports / providers / bootstrap.
- `forRoot` (providers once) vs `forChild` (routes/declarables).
- Know the legacy → modern mapping table.
- Migrate incrementally with official schematics.

## Readiness Checklist

- [ ] Read and explain a legacy NgModule app
- [ ] Explain `forRoot`/`forChild` duplicate-instance problem
- [ ] Convert a legacy component to modern APIs by hand
- [ ] List the migration schematics in order
