# 19 — Architecture & Best Practices 🔥🔥

> **Interview + Understanding Guide** — Roadmap Section 19
>
> Goal: Structure scalable Angular apps — the modern style guide, feature-based folders, smart vs presentational components, layering, monorepos (Nx), micro-frontends (Native Federation), and code-quality conventions. Senior interviews love "how would you architect…?"

---

# 1. Modern Angular Style Guide (v20+) ⭐

The official style guide was simplified in v20:

- **No mandatory type suffixes**: `user-profile.ts` exports `UserProfile` (not `user-profile.component.ts` / `UserProfileComponent`). Services: `user-api.ts` → `UserApi`. CLI generates this by default; existing projects keep their naming via config.
- Keep **one concept per file**; files named in kebab-case after their main export.
- Prefer `inject()` over constructor injection.
- `protected` for members used only by the template; `readonly` for inputs/outputs/queries.
- Use `[class]`/`[style]` bindings over `NgClass`/`NgStyle`.
- Use `host` metadata instead of `@HostBinding`/`@HostListener`.
- Name event handlers for what they **do** (`saveUser()`), not the event (`onClick()`).
- Keep lifecycle hooks simple — delegate to well-named methods.
- Group related files in a folder by feature.

---

# 2. Folder Structure — Feature-Based ⭐⭐

```text
src/app/
├── core/                      # app-wide singletons: auth, interceptors, error handler, layout shell
│   ├── auth/
│   ├── http/                  # interceptors
│   └── layout/
├── shared/                    # reusable, stateless UI & utils (no feature knowledge)
│   ├── ui/                    # button, card, modal, spinner
│   ├── pipes/
│   ├── directives/
│   └── utils/
├── features/
│   ├── products/
│   │   ├── data-access/       # api services, stores, models
│   │   ├── feature-list/      # smart/container pages
│   │   ├── feature-detail/
│   │   ├── ui/                # presentational components for this feature
│   │   └── products.routes.ts # lazy routes
│   └── checkout/
├── app.config.ts
├── app.routes.ts
└── app.ts
```

Principles:
- **Organize by feature (domain), not by type** (avoid giant `components/`, `services/` folders).
- Each feature is **lazy-loaded** via its routes file.
- **Dependency rules**: features can use `shared` and `core`; `shared` never imports features; features don't import each other's internals (go through a public API / `index.ts`).
- Library types (Nx convention): **feature**, **ui**, **data-access**, **util**.

---

# 3. Smart (Container) vs Presentational (Dumb) Components ⭐⭐

| Smart / container | Presentational / dumb |
| --- | --- |
| Knows about services, stores, routing | Knows nothing about where data comes from |
| Fetches data, handles actions | Gets data via **inputs**, emits **outputs** |
| Few, at route level | Many, reusable |
| Harder to reuse | Easy to test & reuse, OnPush-friendly |

```ts
// smart
@Component({
  selector: "app-products-page",
  imports: [ProductGrid],
  template: `<app-product-grid [products]="store.visible()" (addToCart)="cart.add($event)" />`,
})
export class ProductsPage {
  protected readonly store = inject(ProductStore);
  protected readonly cart = inject(CartStore);
}

// presentational
@Component({
  selector: "app-product-grid",
  template: `@for (p of products(); track p.id) { <app-product-card [product]="p" (add)="addToCart.emit(p)" /> }`,
})
export class ProductGrid {
  readonly products = input.required<Product[]>();
  readonly addToCart = output<Product>();
}
```

---

# 4. Layering

```text
UI (components, templates)
  ↓ uses
State / facade (signal stores, NgRx, facades)
  ↓ uses
Data access (API services, HttpClient, mappers DTO → model)
  ↓
Backend
```

- Components don't call `HttpClient` directly — go through services/stores.
- Map API DTOs to domain models at the boundary.
- A **facade** service can hide the state implementation (signals vs NgRx) from components.

---

# 5. Monorepos with Nx ⭐

- Multiple apps + libraries in one repo; shared code as libraries.
- **Module boundary rules** (tags like `scope:products`, `type:ui`) enforced by lint.
- **Affected** commands — only test/build what changed; computation caching (local/remote).
- Generators for consistent code; project graph visualization.

---

# 6. Micro-Frontends ⭐

Split a large frontend into independently deployable apps owned by different teams.

| Approach | Notes |
| --- | --- |
| **Native Federation** (`@angular-architects/native-federation`) | Module-federation concept on top of esbuild/import maps — the recommended path with the Angular CLI's esbuild builder |
| Module Federation (webpack) | Classic; webpack builders deprecated in v22 |
| Web Components (`@angular/elements`) | Framework-agnostic integration |
| iframes / route-level composition | Strong isolation, weaker UX |

Trade-offs: team autonomy and independent deploys vs complexity, shared dependency versions, consistent UX, performance overhead. Use only when organizational scale justifies it.

---

# 7. Code Quality

- **Strict TypeScript** + `strictTemplates` (on by default in v22)
- **ESLint** (`angular-eslint`) + Prettier; template accessibility rules
- Conventional commits, husky + lint-staged
- Unit + component + E2E tests in CI; bundle budgets
- `ng update` regularly (yearly majors from v22; stay within support window)
- Environment config via `environment.ts` / runtime config loaded with `provideAppInitializer`
- Global error handling: custom `ErrorHandler` + `provideBrowserGlobalErrorListeners()` + monitoring (Sentry)
- i18n: `@angular/localize` (build-time) or Transloco/ngx-translate (runtime)

```ts
@Service()
export class GlobalErrorHandler implements ErrorHandler {
  private readonly monitoring = inject(MonitoringService);
  handleError(error: unknown) {
    this.monitoring.capture(error);
    console.error(error);
  }
}
// providers: [{ provide: ErrorHandler, useClass: GlobalErrorHandler }]
```

---

# 8. Modern Angular Best Practices Cheat Sheet ⭐⭐⭐

```text
✅ Standalone components (default), no new NgModules
✅ Signals for state; computed for derived; effects only for side effects
✅ input()/output()/model()/viewChild() signal APIs
✅ OnPush (default v22) + zoneless (default v21)
✅ Built-in control flow (@if/@for/@switch) with track
✅ inject() and @Service()
✅ Functional guards/resolvers/interceptors
✅ Lazy routes + @defer
✅ httpResource for reads; HttpClient for writes
✅ Signal Forms (new) / typed reactive forms
✅ NgOptimizedImage
✅ SSR + incremental hydration where SEO/LCP matter
✅ Vitest, harnesses/testing library
✅ Feature folders, smart/presentational split
```

---

# 9. Interview Answers

**How would you structure a large Angular application?**

> "By feature: each domain — products, checkout, account — is a folder or Nx library with lazy routes, split into data-access (API services and stores), feature (smart route components) and ui (presentational components). Shared holds stateless reusable UI and utilities; core holds app-wide singletons like auth, interceptors, error handling and the layout shell. I enforce dependency rules — features don't reach into each other, shared doesn't depend on features — with lint boundaries. Components get data from stores or facades, not HttpClient directly. For very large orgs, an Nx monorepo and possibly micro-frontends with Native Federation."

**Smart vs presentational components?**

> "Smart components know about services, state and routing — they fetch data and handle actions, usually at route level. Presentational components only receive inputs and emit outputs, with no knowledge of where data comes from, so they're reusable, easy to test and ideal for OnPush. It keeps business logic in few places and UI in many small, predictable pieces."

**When would you use micro-frontends?**

> "When multiple teams need to develop and deploy parts of a large frontend independently. With Angular I'd use Native Federation on the esbuild builder. The cost is complexity — shared dependency versions, consistent design, performance and integration testing — so for a single team a well-structured monorepo is usually better."

---

# 10. Key Takeaways

- v20 style guide: no suffixes, `inject()`, `protected` template members, `host` metadata.
- Feature-based folders; core/shared/features; boundaries.
- Smart vs presentational; layering with data-access/facades.
- Nx for scale; Native Federation for micro-frontends.
- Strictness, linting, testing, budgets, regular updates.

## Readiness Checklist

- [ ] Sketch the folder structure of an e-commerce app
- [ ] Split a page into smart + presentational components
- [ ] Explain Nx boundaries and affected builds
- [ ] Discuss micro-frontend trade-offs
