# 01 — Angular Fundamentals & Architecture

> **Interview + Understanding Guide** — Roadmap Section 1
>
> Goal: Understand what Angular is, how an Angular app is structured and bootstrapped today (standalone, signals, zoneless), and how the CLI and build system work.

---

# 1. What is Angular?

**Angular** is a **TypeScript-based, full-featured front-end framework** maintained by Google for building single-page and server-rendered web applications.

> **Interview definition:** "Angular is an opinionated, batteries-included framework for building web apps with TypeScript. It provides components with templates, a reactive model based on signals (and RxJS), dependency injection, routing, forms, HTTP, testing, SSR and a CLI — all maintained together with a consistent release schedule."

### Framework vs library

| Angular (framework) | React (library) |
| --- | --- |
| Routing, forms, HTTP, DI, testing, SSR, i18n, animations included | View layer only; you pick router/state/forms |
| Strong conventions & CLI | Flexible, many ways to do things |
| TypeScript first | JS or TS |
| HTML templates with Angular syntax | JSX |

### AngularJS vs Angular

| AngularJS (1.x, 2010) | Angular (2+, 2016 →) |
| --- | --- |
| JavaScript, MVC, `$scope`, controllers | TypeScript, component-based |
| Two-way binding with digest cycle (watchers) | Unidirectional data flow, explicit two-way binding |
| End of life since Jan 2022 | Actively developed (v22 in 2026) |

"Angular" today always means Angular 2+.

### Release model

- Semantic versioning; **major releases now yearly** (v22 in June 2026, v23 planned for June 2027; previously every 6 months).
- Each major: ~6 months+ active support then LTS (security fixes) — check angular.dev/reference/releases.
- Upgrades are automated with `ng update` migrations (schematics).

---

# 2. Architecture Overview ⭐

```text
┌──────────────────────────────── Angular Application ────────────────────────────────┐
│                                                                                      │
│  bootstrapApplication(App, appConfig)                                                │
│         │                                                                            │
│         ▼                                                                            │
│   Root Component (App) ── template ──▶ child components ──▶ …  (component tree)      │
│         │                     │                                                      │
│         │              bindings, @if/@for, pipes, directives                         │
│         │                                                                            │
│   Signals (state) ◀──── Services (business logic, data) ◀──── HttpClient / resources  │
│         ▲                     ▲                                                      │
│         └──── Dependency Injection provides services to components ────┘              │
│                                                                                      │
│   Router (URL ↔ components) · Forms · Change detection · SSR/hydration               │
└──────────────────────────────────────────────────────────────────────────────────────┘
```

| Building block | Role |
| --- | --- |
| **Component** | Class + template + styles; a piece of UI |
| **Template** | HTML with Angular syntax (bindings, control flow, pipes) |
| **Signals** | Reactive state primitives (`signal`, `computed`, `effect`) |
| **Service** | Reusable logic/data access, provided via DI |
| **Dependency Injection** | Supplies dependencies (services, config) to classes |
| **Directive** | Adds behaviour to elements (attribute) or changes structure |
| **Pipe** | Transforms values in templates (`date`, `currency`, custom) |
| **Router** | Maps URLs to components, lazy loading, guards |
| **Forms** | Template-driven, Reactive, and Signal Forms |
| **HttpClient / resources** | Server communication |
| **NgModule** (legacy) | Former unit of organization; replaced by standalone components |

---

# 3. Angular CLI ⭐

```bash
npm install -g @angular/cli
ng new shop                         # prompts: styles, SSR, AI tooling config…
cd shop
ng serve                            # dev server (Vite-based) → http://localhost:4200
ng build                            # production build → dist/
ng test                             # unit tests (Vitest by default for new projects since v21)
ng e2e                              # end-to-end (after adding a runner, e.g. Playwright)
ng lint                             # after `ng add angular-eslint`
ng update                           # upgrade Angular with automatic migrations
ng add @angular/material            # add libraries with schematics
ng add @angular/ssr                 # add server-side rendering
```

### Generators

```bash
ng generate component features/product-list   # ng g c
ng generate service core/auth                  # ng g s   (v22: generates @Service())
ng generate directive shared/highlight         # ng g d
ng generate pipe shared/truncate               # ng g p
ng generate guard core/auth                    # ng g g   (functional guard)
ng generate interceptor core/auth              # functional interceptor
ng generate interface models/user
ng generate environments
```

Useful flags: `--dry-run`, `--skip-tests`, `--inline-template`, `--inline-style`.

### Style guide naming (v20+)

New projects follow the updated style guide: files and classes **without type suffixes** by default — `product-list.ts` exporting `ProductList` (instead of `product-list.component.ts` / `ProductListComponent`). Both styles appear in real code; be consistent within a project.

---

# 4. Project Structure

```text
shop/
├── angular.json            # workspace/build configuration (builders, budgets, assets)
├── package.json
├── tsconfig.json           # strict TS + strictTemplates
├── public/                 # static assets copied as-is (favicon, images)
└── src/
    ├── index.html          # host page with <app-root>
    ├── main.ts             # bootstraps the app
    ├── main.server.ts      # (SSR) server bootstrap
    ├── server.ts           # (SSR) Node/Express server
    ├── styles.css          # global styles
    └── app/
        ├── app.ts          # root component
        ├── app.html
        ├── app.css
        ├── app.config.ts   # application-wide providers
        ├── app.routes.ts   # route definitions
        └── features/…      # your feature folders
```

---

# 5. Bootstrapping (Standalone) ⭐

```ts
// src/main.ts
import { bootstrapApplication } from "@angular/platform-browser";
import { appConfig } from "./app/app.config";
import { App } from "./app/app";

bootstrapApplication(App, appConfig).catch((err) => console.error(err));
```

```ts
// src/app/app.config.ts
import { ApplicationConfig, provideBrowserGlobalErrorListeners } from "@angular/core";
import { provideRouter, withComponentInputBinding } from "@angular/router";
import { provideHttpClient, withInterceptors } from "@angular/common/http";
import { routes } from "./app.routes";
import { authInterceptor } from "./core/auth.interceptor";

export const appConfig: ApplicationConfig = {
  providers: [
    provideBrowserGlobalErrorListeners(),
    provideRouter(routes, withComponentInputBinding()),
    provideHttpClient(withInterceptors([authInterceptor])),  // uses Fetch by default (v22)
    // New apps are zoneless by default (v21+). For zone.js apps: provideZoneChangeDetection({ eventCoalescing: true })
  ],
};
```

```ts
// src/app/app.ts — root component
import { Component, signal } from "@angular/core";
import { RouterOutlet, RouterLink } from "@angular/router";

@Component({
  selector: "app-root",
  imports: [RouterOutlet, RouterLink],
  template: `
    <header>
      <a routerLink="/">Home</a>
      <a routerLink="/products">Products</a>
    </header>
    <h1>{{ title() }}</h1>
    <router-outlet />
  `,
})
export class App {
  protected readonly title = signal("Shop");
}
```

```html
<!-- src/index.html -->
<body>
  <app-root></app-root>
</body>
```

### What happens at startup

```text
1. Browser loads index.html + main JS bundle
2. bootstrapApplication creates the root environment injector from appConfig.providers
3. Angular creates the root component, finds <app-root> in index.html, renders its template
4. Router (if provided) matches the URL and renders routed components into <router-outlet>
5. Change detection keeps the DOM in sync with state (signals / events)
```

---

# 6. Standalone Components (the default)

Since **v19**, components, directives and pipes are **standalone by default** (no `standalone: true` needed). Each component declares its own dependencies in `imports`.

```ts
@Component({
  selector: "app-product-card",
  imports: [CurrencyPipe, RouterLink, Rating],   // what this template uses
  template: `...`,
})
export class ProductCard {}
```

NgModules still exist (for libraries and legacy apps) — see `20-ngmodules-and-legacy.md`.

**Why standalone?** Less boilerplate, clearer dependencies, easier lazy loading, better tree shaking, simpler mental model and testing.

---

# 7. Compilation & Build ⭐

### Ahead-of-Time (AOT) compilation

Angular compiles templates into efficient JavaScript instructions **at build time** (the **Ivy** compiler/runtime, default since v9).

| AOT (default) | JIT (legacy dev mode) |
| --- | --- |
| Templates compiled at build time | Compiled in the browser at runtime |
| Smaller bundles (no compiler shipped), faster startup | Bigger, slower |
| Template errors caught at build time | Errors at runtime |
| Better security (no runtime template evaluation) | |

**Ivy** benefits: locality (each component compiled independently), tree-shakable runtime, smaller bundles, faster builds, better debugging, enables standalone and signals features.

### Build system

- Modern **application builder** uses **esbuild** for builds and **Vite** for the dev server (default since v17). Webpack-based builders are **deprecated** (v22).
- Production builds: AOT, minification, tree shaking, code splitting (lazy routes, `@defer`), hashing, budgets.

```json
// angular.json (excerpt) — bundle size budgets fail the build when exceeded
"budgets": [
  { "type": "initial", "maximumWarning": "500kB", "maximumError": "1MB" },
  { "type": "anyComponentStyle", "maximumWarning": "4kB" }
]
```

### Environments

```bash
ng generate environments   # creates src/environments/environment(.development).ts + fileReplacements
```

```ts
// environment.ts
export const environment = { production: true, apiUrl: "https://api.example.com" };
```

---

# 8. Angular's Key Design Principles

- **Components + templates** — declarative UI.
- **Unidirectional data flow** — data flows down via inputs; events up via outputs.
- **Fine-grained reactivity with signals** — Angular knows exactly what changed.
- **Dependency Injection everywhere** — testable, decoupled code.
- **TypeScript & strict templates** — type errors in templates caught at build time (`strictTemplates` default in v22).
- **Batteries included** — consistent first-party solutions.
- **Automated upgrades** — `ng update` migrations.

---

# 9. Interview Answers

**What is Angular and how does it differ from React?**

> "Angular is a complete TypeScript framework from Google: components with HTML templates, signals and RxJS for reactivity, built-in dependency injection, router, forms, HTTP client, testing setup, SSR and a CLI. React is a UI library — you choose routing, state, forms and build tools. Angular is more opinionated and consistent across teams, which suits large enterprise apps; React is more flexible."

**What happens when an Angular app starts?**

> "`main.ts` calls `bootstrapApplication` with the root component and an application config. Angular creates the root environment injector from the configured providers — router, HTTP client, etc. — creates the root component, renders it into its selector in `index.html`, the router renders the matched route into `router-outlet`, and change detection keeps the view in sync."

**What is AOT compilation?**

> "Ahead-of-time compilation converts templates and components into optimized JavaScript during the build, so the browser doesn't need the Angular compiler. That means smaller bundles, faster rendering, template errors caught at build time, and better security. It's the default with the Ivy engine."

**What are standalone components?**

> "Components, directives and pipes that declare their own template dependencies in an `imports` array instead of being declared in an NgModule. They're the default since v19, reduce boilerplate, make lazy loading and tree shaking easier, and simplify testing."

---

# 10. Key Takeaways

- Angular = full TypeScript framework (components, signals, DI, router, forms, HTTP, SSR, CLI).
- Modern Angular: **standalone**, **signals**, **zoneless** (new apps v21+), **OnPush by default** (v22), **control flow blocks**, **Vitest**, esbuild/Vite.
- Bootstrap with `bootstrapApplication(App, appConfig)`; providers in `app.config.ts`.
- AOT + Ivy compile templates at build time.
- CLI generates code and automates upgrades.

## Readiness Checklist

- [ ] Explain Angular's building blocks and draw the architecture
- [ ] Create a project, generate a component/service, run/build/test
- [ ] Explain bootstrapping and `app.config.ts`
- [ ] Explain AOT/Ivy and standalone components
