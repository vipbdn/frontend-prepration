# 10 — Routing 🔥🔥

> **Interview + Understanding Guide** — Roadmap Section 10
>
> Goal: Build and explain Angular routing — route config, params, nested/child routes, lazy loading, preloading, functional guards and resolvers, route input binding, navigation, and v22 router changes.

---

# 1. Setup ⭐

```ts
// app.routes.ts
import { Routes } from "@angular/router";

export const routes: Routes = [
  { path: "", component: Home, title: "Home" },
  { path: "products", component: ProductList, title: "Products" },
  { path: "products/:id", component: ProductDetail },
  { path: "old-products", redirectTo: "products", pathMatch: "full" },
  { path: "**", component: NotFound },          // wildcard — must be LAST
];

// app.config.ts
provideRouter(
  routes,
  withComponentInputBinding(),                   // params → inputs
  withViewTransitions(),                         // View Transitions API
  withInMemoryScrolling({ scrollPositionRestoration: "enabled", anchorScrolling: "enabled" }),
  withPreloading(PreloadAllModules),
);
```

```html
<!-- app.html -->
<nav>
  <a routerLink="/" routerLinkActive="active" [routerLinkActiveOptions]="{ exact: true }">Home</a>
  <a routerLink="/products" routerLinkActive="active" ariaCurrentWhenActive="page">Products</a>
  <a [routerLink]="['/products', product.id]" [queryParams]="{ tab: 'reviews' }">Details</a>
</nav>
<router-outlet />
```

- Routes are matched **in order, first match wins** → specific before generic, `**` last.
- `pathMatch: 'full'` for empty-path redirects (otherwise `''` prefix-matches everything).
- `title` sets `document.title` (custom `TitleStrategy` for patterns like "X | MyShop").

---

# 2. Reading Route Parameters ⭐

## 2.1 Input binding (modern)

```ts
// route: products/:id   URL: /products/42?tab=reviews
export class ProductDetail {
  readonly id = input.required<string>();     // path param
  readonly tab = input<string>();             // query param
  readonly product = input<Product>();        // resolver data key
  protected readonly data = httpResource<Product>(() => `/api/products/${this.id()}`);
}
```

🆕 **v22:** `withComponentInputBinding({ queryParams: false, unmatchedInputBehavior: … })` lets you configure which sources bind and what happens to inputs with no matching param.

## 2.2 `ActivatedRoute` (classic)

```ts
export class ProductDetailLegacy {
  private readonly route = inject(ActivatedRoute);

  readonly id = this.route.snapshot.paramMap.get("id");            // snapshot: one-time read
  readonly id$ = this.route.paramMap.pipe(map((p) => p.get("id")!)); // observable: updates when reused
  readonly tab$ = this.route.queryParamMap.pipe(map((q) => q.get("tab")));
  readonly data$ = this.route.data;                                  // static data + resolvers
}
```

⚠️ **Snapshot pitfall:** navigating `/products/1 → /products/2` **reuses** the same component instance, so a snapshot read in `ngOnInit` goes stale. Use the observable, an input (signal), or `toSignal(route.paramMap)`.

---

# 3. Navigation

```ts
private readonly router = inject(Router);

this.router.navigate(["/products", id]);
this.router.navigate(["../edit"], { relativeTo: this.route });
this.router.navigate([], { queryParams: { page: 2 }, queryParamsHandling: "merge" });   // keep other params
this.router.navigateByUrl("/checkout", { replaceUrl: true, state: { from: "cart" } });
```

`queryParamsHandling`: `'merge'` | `'preserve'` | `''`. Both return `Promise<boolean>`.

🆕 v22: `RouterLink` `browserUrl` input — show a different URL in the address bar than the one routed.

---

# 4. Child (Nested) Routes ⭐

```ts
{
  path: "account",
  component: AccountLayout,              // contains its own <router-outlet />
  children: [
    { path: "", redirectTo: "profile", pathMatch: "full" },
    { path: "profile", component: Profile },
    { path: "orders", component: Orders },
    { path: "orders/:orderId", component: OrderDetail },
  ],
}
```

🆕 **v22:** `paramsInheritanceStrategy` now defaults to **`'always'`** — child routes inherit parent params & data (previously only for componentless/empty-path parents). No migration; check code that assumed otherwise.

Named outlets: `<router-outlet name="sidebar" />` + `{ path: "help", component: Help, outlet: "sidebar" }`.

---

# 5. Lazy Loading ⭐⭐⭐

Load code only when the route is visited → smaller initial bundle.

```ts
export const routes: Routes = [
  // lazy single component
  { path: "about", loadComponent: () => import("./about/about").then((m) => m.About) },

  // with default export: loadComponent: () => import("./about/about")

  // lazy group of child routes
  { path: "admin", loadChildren: () => import("./admin/admin.routes").then((m) => m.ADMIN_ROUTES) },
];

// admin/admin.routes.ts
export const ADMIN_ROUTES: Routes = [
  {
    path: "",
    providers: [AdminApi],                  // route-level providers (own environment injector)
    children: [
      { path: "", component: AdminDashboard },
      { path: "users", component: AdminUsers },
    ],
  },
];
```

Legacy: `loadChildren: () => import("./admin/admin.module").then(m => m.AdminModule)`.

## Preloading strategies

| Strategy | Behaviour |
| --- | --- |
| `NoPreloading` (default) | Load lazy chunks on navigation |
| `PreloadAllModules` | After initial load, download all lazy routes in background |
| Custom | e.g. preload routes with `data: { preload: true }`, or on network idle / hover (ngx-quicklink) |

```ts
@Service()
export class SelectivePreloading implements PreloadingStrategy {
  preload(route: Route, load: () => Observable<unknown>) {
    return route.data?.["preload"] ? load() : of(null);
  }
}
// provideRouter(routes, withPreloading(SelectivePreloading))
```

Component-level lazy loading: `@defer` (see `03-control-flow-and-defer.md`).

---

# 6. Guards ⭐⭐⭐ (functional)

| Guard | Question | Typical use |
| --- | --- | --- |
| `canActivate` | Can the user enter this route? | Auth, roles |
| `canActivateChild` | Can enter child routes? | Protect a whole section |
| `canDeactivate` | Can the user leave? | Unsaved changes |
| `canMatch` | Should this route even **match**? (else try next route) | Role-based alternate routes, feature flags; prevents lazy chunk download |
| `resolve` | Fetch data before activating | Preload required data |

Return `boolean | UrlTree | RedirectCommand` (or Observable/Promise of these).

```ts
import { CanActivateFn, CanMatchFn, CanDeactivateFn, Router } from "@angular/router";

export const authGuard: CanActivateFn = (route, state) => {
  const auth = inject(AuthStore);
  const router = inject(Router);
  return auth.isLoggedIn() ? true : router.createUrlTree(["/login"], { queryParams: { returnUrl: state.url } });
};

export const roleGuard = (role: string): CanActivateFn => () =>
  inject(AuthStore).roles().includes(role) || inject(Router).parseUrl("/forbidden");

// v22: canMatch receives a required 3rd param — the current router snapshot
export const adminMatch: CanMatchFn = (route, segments, currentSnapshot) => inject(AuthStore).isAdmin();

export interface HasUnsavedChanges { hasUnsavedChanges(): boolean }
export const unsavedChangesGuard: CanDeactivateFn<HasUnsavedChanges> = (component) =>
  !component.hasUnsavedChanges() || confirm("Discard unsaved changes?");
```

```ts
{ path: "admin", canMatch: [adminMatch], loadChildren: () => import("./admin/admin.routes").then(m => m.ADMIN_ROUTES) },
{ path: "admin", component: AccessDenied },                // fallback if canMatch fails
{ path: "settings", canActivate: [authGuard], canDeactivate: [unsavedChangesGuard], component: Settings },
{ path: "reports", canActivate: [authGuard, roleGuard("analyst")], component: Reports },
```

### `canActivate` vs `canMatch` (interview favourite)

- `canActivate`: route **matched**, lazy code **already downloaded**, then access is checked.
- `canMatch`: decides whether the route matches at all — if false, the router **tries the next route** and **doesn't download** the lazy chunk. Better for role-based loading.

⚠️ Guards are **UX, not security** — always enforce authorization on the server.

Class-based guards (`implements CanActivate`) are deprecated in favour of functions.

---

# 7. Resolvers

```ts
export const productResolver: ResolveFn<Product> = (route) =>
  inject(ProductApi).get(route.paramMap.get("id")!);

{ path: "products/:id", component: ProductDetail, resolve: { product: productResolver } }
// → available as input `product` (with input binding) or route.data
```

Trade-off: the navigation **waits** for data (blank screen feel). Many apps prefer navigating immediately and showing a skeleton (resource/`@defer`). Handle errors in resolvers (redirect or fallback) or navigation fails.

---

# 8. Router Events & Loading Bar

```ts
readonly navigating = toSignal(
  inject(Router).events.pipe(
    filter((e) => e instanceof NavigationStart || e instanceof NavigationEnd || e instanceof NavigationCancel || e instanceof NavigationError),
    map((e) => e instanceof NavigationStart),
  ),
  { initialValue: false },
);
```

Sequence: `NavigationStart → RoutesRecognized → GuardsCheckStart/End → ResolveStart/End → ActivationEnd → NavigationEnd` (or Cancel/Error).

---

# 9. Other Features

- **Location strategies:** `PathLocationStrategy` (default, HTML5 `pushState`, needs server fallback to `index.html`) vs `HashLocationStrategy` (`/#/path`, `withHashLocation()`).
- **Route reuse strategy:** customize keeping components alive between routes.
- **`withRouterConfig({ onSameUrlNavigation: "reload" })`**.
- **Redirect functions:** `redirectTo: ({ queryParams }) => …` returning a string/UrlTree.
- **Error handling:** `withNavigationErrorHandler`.

---

# 10. Interview Answers

**How does lazy loading work in Angular?**

> "Instead of importing a component statically, the route uses `loadComponent` or `loadChildren` with a dynamic `import()`. The bundler creates a separate chunk that the router downloads only when the route is first visited, shrinking the initial bundle. I combine it with a preloading strategy — all, selective via route data, or on idle — so navigation stays fast, and with `canMatch` guards so unauthorized users never download admin code."

**What guards exist? canActivate vs canMatch?**

> "`canActivate`, `canActivateChild`, `canDeactivate`, `canMatch`, plus resolvers. Today they're plain functions using `inject()`, returning a boolean, a UrlTree for redirects, or an observable/promise. `canActivate` runs after the route matched and its lazy code loaded; `canMatch` decides whether the route matches at all — if it returns false the router tries the next route and the lazy chunk isn't downloaded. Guards are for UX — real security is on the server."

**Snapshot vs observable params?**

> "The snapshot is a one-time read. When navigating between URLs that use the same route, Angular reuses the component instance, so a snapshot read in `ngOnInit` won't update. Subscribing to `paramMap`, or using route input binding where params become signal inputs, handles that."

**What is a resolver and when would you avoid it?**

> "A function that fetches data before the route activates, exposing it via route data or inputs. It guarantees data on entry, but delays navigation and gives no visual feedback, so for slow APIs I prefer navigating immediately and rendering a skeleton with a resource."

---

# 11. Key Takeaways

- Order matters; `**` last; `pathMatch: 'full'` for empty redirects.
- `withComponentInputBinding` → params/query/data as inputs.
- Lazy: `loadComponent` / `loadChildren`; preloading strategies.
- Functional guards; `canMatch` prevents chunk download; v22 adds its 3rd param.
- v22: `paramsInheritanceStrategy: 'always'` by default.

## Readiness Checklist

- [ ] Configure lazy feature routes with route-level providers
- [ ] Write auth, role, canMatch and canDeactivate guards
- [ ] Bind params to inputs
- [ ] Build a custom preloading strategy
- [ ] Show a navigation loading bar from router events
