# 06 — Dependency Injection 🔥🔥

> **Interview + Understanding Guide** — Roadmap Section 6
>
> Goal: Understand Angular's DI system deeply — what it is, injector hierarchy, providers and their scopes, `inject()`, `InjectionToken`, provider recipes, resolution modifiers — plus the v22 additions `@Service()` and `injectAsync()`.

---

# 1. What is Dependency Injection? ⭐

**Dependency Injection (DI)** is a design pattern where a class **receives** its dependencies from the outside instead of creating them itself. Angular has a built-in DI framework that creates and supplies dependencies (usually services) to components, directives, pipes and other services.

```ts
// ❌ Without DI: tightly coupled, hard to test, hard to swap implementations
export class OrderService {
  private http = new HttpClient(/* …needs its own dependencies… */);
}

// ✅ With DI: Angular provides it
export class OrderService2 {
  private readonly http = inject(HttpClient);
}
```

### Benefits

- **Loose coupling** — depend on abstractions, swap implementations (mock in tests, different config per environment)
- **Testability** — provide fakes via `TestBed`
- **Singletons & lifetime management** — Angular decides when to create/destroy
- **Configuration** — inject config values/tokens
- **Hierarchical scoping** — share instances app-wide or per component subtree

> **Interview definition:** "DI is a pattern where classes declare what they need and an injector supplies it. In Angular, providers tell injectors how to create a dependency, and injectors are hierarchical — environment injectors for app-wide services and element injectors for component-level instances. You request dependencies with `inject()` or constructor parameters."

---

# 2. Creating & Providing Services ⭐

## 2.1 `@Service()` (v22) ⭐

```ts
import { Service, inject } from "@angular/core";
import { HttpClient } from "@angular/common/http";

@Service()                         // = @Injectable({ providedIn: "root" }), tree-shakable singleton
export class ProductService {
  private readonly http = inject(HttpClient);   // @Service classes must use inject()

  getProducts() {
    return this.http.get<Product[]>("/api/products");
  }
}
```

- `ng generate service` produces `@Service()` in v22 (`--injectable` for the old decorator).
- `@Service({ autoProvided: false })` opts out of automatic root provision (you then provide it explicitly).
- No automatic migration — existing `@Injectable` code keeps working.

## 2.2 `@Injectable` (classic — everywhere in existing code)

```ts
@Injectable({ providedIn: "root" })   // singleton in the root injector, tree-shakable
export class AuthService {}

@Injectable()                          // must be provided explicitly in some `providers` array
export class WizardState {}
```

### Where can you provide?

| Location | Scope / lifetime |
| --- | --- |
| `providedIn: "root"` / `@Service()` | **App-wide singleton**, created lazily on first injection, tree-shaken if unused |
| `bootstrapApplication(App, { providers: [...] })` / `app.config.ts` | App-wide (environment injector) |
| Route `providers: [...]` | Shared by that route and its children (lazy routes get their own environment injector) |
| Component/directive `providers: [...]` | **New instance per component instance**, shared with its view & content descendants; destroyed with the component |
| Component `viewProviders: [...]` | Like `providers` but only visible to the component's **view**, not projected content |
| `providedIn: "platform"` | Shared across multiple apps on the page (rare) |

---

# 3. Injecting Dependencies ⭐

## 3.1 `inject()` (preferred)

```ts
export class ProductList {
  private readonly productService = inject(ProductService);
  private readonly route = inject(ActivatedRoute);
  private readonly destroyRef = inject(DestroyRef);
}
```

## 3.2 Constructor injection (classic)

```ts
export class ProductListLegacy {
  constructor(
    private readonly productService: ProductService,
    @Inject(API_URL) private readonly apiUrl: string,
    @Optional() private readonly logger?: Logger,
  ) {}
}
```

### Why `inject()` is preferred

- Works in field initializers; less boilerplate; plays well with inheritance (no `super(...)` parameter juggling)
- Usable in **functions** (functional guards, interceptors, resolvers, custom `injectX()` helpers)
- Better type inference with tokens; doesn't rely on TypeScript decorator metadata
- Required for `@Service()` classes

## 3.3 Injection context ⚠️

`inject()` only works in an **injection context**:
- constructors / field initializers of DI-created classes
- factory functions (`useFactory`, `InjectionToken` factory)
- functional guards, resolvers, interceptors
- inside `runInInjectionContext(injector, fn)`

```ts
export class Report {
  private readonly injector = inject(Injector);

  later() {
    // inject(HttpClient) here would throw NG0203 (not in injection context)
    runInInjectionContext(this.injector, () => {
      const http = inject(HttpClient);
    });
  }
}
```

### Reusable inject helpers

```ts
export function injectCurrentUser() {
  return inject(AuthService).user;        // returns a signal
}

export class Profile {
  protected readonly user = injectCurrentUser();
}
```

---

# 4. Providers & Recipes ⭐⭐⭐

A **provider** tells an injector **how to create** a value for a **token**.

```ts
providers: [
  ProductService,                                         // shorthand for { provide: ProductService, useClass: ProductService }

  { provide: Logger, useClass: ConsoleLogger },           // useClass: substitute an implementation
  { provide: Logger, useClass: environment.production ? RemoteLogger : ConsoleLogger },

  { provide: API_URL, useValue: "https://api.example.com" }, // useValue: constants/config objects

  {                                                       // useFactory: computed at creation, can inject
    provide: StorageService,
    useFactory: () => (inject(PLATFORM_ID) === "browser" ? new LocalStorageService() : new MemoryStorageService()),
  },

  { provide: OldLogger, useExisting: Logger },            // useExisting: alias → same instance as Logger
]
```

| Recipe | Creates | Use case |
| --- | --- | --- |
| `useClass` | `new SomeClass()` | Swap implementations, mocks |
| `useValue` | The given value | Config, constants, test doubles |
| `useFactory` | Result of a function | Conditional creation, needs runtime info |
| `useExisting` | Alias to another token | Expose one service under multiple tokens/interfaces |

### `useClass` vs `useExisting` (interview favourite)

```ts
{ provide: A, useClass: B }     // a NEW instance of B for token A (separate from any B instance)
{ provide: A, useExisting: B }  // token A resolves to the SAME instance as token B
```

## 4.1 `InjectionToken` — for non-class dependencies ⭐

Interfaces don't exist at runtime, and strings as tokens collide. Use `InjectionToken`:

```ts
import { InjectionToken, inject } from "@angular/core";

export interface AppConfig { apiUrl: string; featureFlags: Record<string, boolean> }

export const APP_CONFIG = new InjectionToken<AppConfig>("APP_CONFIG");

// Token with a default factory (tree-shakable, providedIn root)
export const WINDOW = new InjectionToken<Window>("WINDOW", {
  providedIn: "root",
  factory: () => window,
});

// provide
bootstrapApplication(App, {
  providers: [{ provide: APP_CONFIG, useValue: { apiUrl: "/api", featureFlags: { newCheckout: true } } }],
});

// inject
export class Api {
  private readonly config = inject(APP_CONFIG);   // typed as AppConfig
}
```

## 4.2 Multi providers

Multiple values for the same token, injected as an **array**:

```ts
export const VALIDATION_RULES = new InjectionToken<Rule[]>("VALIDATION_RULES");

providers: [
  { provide: VALIDATION_RULES, useClass: RequiredRule, multi: true },
  { provide: VALIDATION_RULES, useClass: EmailRule, multi: true },
]

const rules = inject(VALIDATION_RULES);   // [RequiredRule, EmailRule]
```

Angular uses this for `HTTP_INTERCEPTORS` (class interceptors), `NG_VALIDATORS`, `NG_VALUE_ACCESSOR`, `APP_INITIALIZER` (now `provideAppInitializer`).

## 4.3 App initialization

```ts
providers: [
  provideAppInitializer(() => {
    const config = inject(ConfigService);
    return config.load();          // app waits for the returned promise/observable before bootstrapping
  }),
]
```

---

# 5. Injector Hierarchy ⭐⭐⭐

Angular has **two injector hierarchies**:

```text
NullInjector  (throws NG0201 "No provider for X")
     ▲
Platform injector          (providedIn: "platform")
     ▲
Root environment injector  (providedIn: "root", app.config providers)
     ▲
Lazy route environment injectors (route providers, lazy-loaded routes)
     ▲
─────────────────────────────────────────────────────────────
Element injectors (component/directive `providers`) — follow the DOM/component tree:
  AppComponent injector
     ▲
  ProductList injector
     ▲
  ProductCard injector
```

### Resolution algorithm

When a component injects a token:
1. Look in **its own element injector**, then **parent element injectors** up the component tree.
2. If not found, look in the **environment injector** hierarchy (route → root → platform).
3. If still not found → `NullInjector` → error (unless optional).

The **first match wins**, so a provider closer to the component **shadows** ones above it.

### Example: per-component instance

```ts
@Component({
  selector: "app-counter",
  providers: [CounterState],       // each <app-counter> gets its own CounterState
  template: `<app-counter-display /><app-counter-buttons />`,
})
export class Counter {}
```

Both children share **this** counter's `CounterState`; another `<app-counter>` elsewhere has a separate one.

### Lazy-loaded routes ⚠️

A service provided in a lazy route's `providers` gets a **separate instance** from the root one — a classic bug when people expect a singleton. Use `providedIn: "root"` / `@Service()` for true singletons.

---

# 6. Resolution Modifiers

```ts
inject(Logger, { optional: true });   // null instead of throwing if not found
inject(Logger, { self: true });       // only this element's injector
inject(Logger, { skipSelf: true });   // start searching from the parent
inject(Logger, { host: true });       // stop at the host component's injector
```

Decorator equivalents: `@Optional()`, `@Self()`, `@SkipSelf()`, `@Host()`.

```ts
// Prevent re-providing a singleton module/service
export class CoreGuard {
  constructor() {
    const parent = inject(CoreGuard, { optional: true, skipSelf: true });
    if (parent) throw new Error("CoreGuard already loaded — import it only once.");
  }
}
```

---

# 7. `injectAsync()` — lazy-loaded services (v22) 🆕

Code-split heavy services and load them only when needed:

```ts
import { injectAsync } from "@angular/core";

export class ReportsPage {
  private readonly pdfExporter = injectAsync(
    () => import("./pdf-exporter.service").then((m) => m.PdfExporter),
    { prefetch: "onIdle" },          // optionally download the chunk when the browser is idle
  );

  async export() {
    const exporter = await this.pdfExporter();   // loads chunk (if needed) + creates via DI
    exporter.export();
  }
}
```

Use for large dependencies (PDF/Excel generation, charting engines, rich editors) used by a few interactions.

---

# 8. `DestroyRef` & cleanup

```ts
export class LiveTicker {
  constructor() {
    const socket = new WebSocket("wss://example.com/ticker");
    inject(DestroyRef).onDestroy(() => socket.close());   // works in components, directives, services
  }
}
```

`takeUntilDestroyed()` (RxJS interop) uses `DestroyRef` under the hood.

---

# 9. Tree-Shakable Providers

`providedIn: "root"` / `@Service()` / `InjectionToken` with `factory` are **tree-shakable** — if nothing injects the service, it isn't included in the bundle. Services listed in `providers` arrays are always included.

---

# 10. Testing with DI

```ts
TestBed.configureTestingModule({
  providers: [
    { provide: ProductService, useValue: { getProducts: () => of([{ id: "1", name: "Pen", price: 10 }]) } },
    { provide: APP_CONFIG, useValue: { apiUrl: "/test", featureFlags: {} } },
  ],
});
const service = TestBed.inject(ProductService);
```

See `16-testing.md`.

---

# 11. Common Errors

| Error | Cause | Fix |
| --- | --- | --- |
| `NG0201: No provider for X` | Not provided anywhere reachable | Use `providedIn: "root"`/`@Service()` or add to `providers` |
| `NG0203: inject() must be called from an injection context` | `inject()` in a method/callback | Move to field/constructor, or `runInInjectionContext` |
| `NG0200: Circular dependency in DI` | A injects B, B injects A | Refactor; extract shared logic; lazy `inject` inside methods via `Injector` |
| Two instances of a "singleton" | Provided in a lazy route/component as well as root | Provide in one place |

---

# 12. Interview Answers

**Explain Angular's dependency injection.**

> "Angular's DI framework creates and supplies dependencies. A provider maps a token — usually a class, or an `InjectionToken` for non-class values — to a recipe: `useClass`, `useValue`, `useFactory` or `useExisting`. Consumers request a token with `inject()` or a constructor parameter. Injectors are hierarchical: environment injectors (root, and one per lazy route) and element injectors created by component providers. Resolution walks up the element tree, then the environment hierarchy, and the closest provider wins."

**What does `providedIn: 'root'` do? What is `@Service()`?**

> "It registers the service with the root injector as a lazily created, tree-shakable singleton — if nothing injects it, it's dropped from the bundle. In Angular 22, `@Service()` is a shorter decorator for exactly that, and it requires using `inject()` rather than constructor injection."

**How do you get a separate service instance per component?**

> "Provide it in the component's `providers` array. Each component instance creates its own element injector with a new instance, shared by its children and destroyed with the component — useful for component-scoped state like a wizard or a form."

**What is an InjectionToken and why use it?**

> "A unique, typed token for dependencies that aren't classes — configuration objects, primitives, functions, interfaces. Interfaces don't exist at runtime and string tokens can collide, so `new InjectionToken<AppConfig>('APP_CONFIG')` gives a safe key. It can also have a default factory to make it tree-shakable."

**`useClass` vs `useExisting`?**

> "`useClass` creates a new instance of the given class for that token. `useExisting` makes the token an alias that resolves to the same instance as another token."

**Why prefer `inject()` over constructor injection?**

> "It works in field initializers and plain functions — guards, interceptors, resolvers, reusable `injectX` helpers — avoids constructor boilerplate and inheritance issues, gives better typing with tokens, and doesn't rely on decorator metadata. The trade-off is that it must run in an injection context."

---

# 13. Key Takeaways

- DI decouples classes from how dependencies are created; essential for testing.
- `@Service()` (v22) / `@Injectable({ providedIn: "root" })` = tree-shakable singletons.
- Provide at root, route, or component level — scope and lifetime differ.
- Recipes: `useClass`, `useValue`, `useFactory`, `useExisting`, `multi`; `InjectionToken` for non-classes.
- Resolution: element injectors up the tree → environment injectors → NullInjector.
- `inject()` requires an injection context; `injectAsync()` lazy-loads services.

## Readiness Checklist

- [ ] Draw both injector hierarchies and the lookup path
- [ ] Demonstrate root vs component-level instances
- [ ] Provide config with an `InjectionToken`
- [ ] Explain all provider recipes and multi providers
- [ ] Explain the lazy-route double-instance bug
