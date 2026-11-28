# 22 — Angular Interview Questions (Rapid Revision) 🔥🔥🔥

> **100 most-asked questions with crisp answers.** Read the question, answer aloud, then check. Detailed explanations live in the linked topic files.
>
> Levels: 🟢 basic · 🟡 intermediate · 🔴 advanced/senior

---

## A. Fundamentals (01–02)

**1. 🟢 What is Angular?**
A TypeScript-based, full-featured frontend framework by Google with components, DI, routing, forms, HTTP, testing, SSR and a CLI built in.

**2. 🟢 Angular vs AngularJS?**
AngularJS (1.x, 2010) was JavaScript with controllers/`$scope` and digest-cycle dirty checking. Angular (2+) is a TypeScript rewrite with components, AOT compilation, DI hierarchy, RxJS/signals. EOL for AngularJS was Dec 2021.

**3. 🟢 What is a component?**
A TypeScript class with `@Component` metadata (selector, template, styles, imports) controlling a piece of UI.

**4. 🟢 What is a standalone component?**
A component that declares its own dependencies in `imports` without an NgModule. Default since v19.

**5. 🟢 How does an Angular app bootstrap?**
`main.ts` calls `bootstrapApplication(App, appConfig)`; providers (router, http…) come from `app.config.ts`; Angular renders the root component into `index.html`.

**6. 🟢 Types of data binding?**
Interpolation `{{ }}`, property `[prop]`, event `(event)`, two-way `[(ngModel)]`/`[(model)]`, attribute `[attr.x]`, class `[class.x]`, style `[style.x]`.

**7. 🟢 Lifecycle hooks in order?**
constructor → `ngOnChanges` → `ngOnInit` → `ngDoCheck` → `ngAfterContentInit` → `ngAfterContentChecked` → `ngAfterViewInit` → `ngAfterViewChecked` → `ngOnDestroy`. Plus `afterNextRender`/`afterEveryRender`.

**8. 🟢 constructor vs ngOnInit?**
Constructor: class creation, DI. `ngOnInit`: after first inputs are set — initialization logic using inputs. With signal inputs, `computed`/`effect` often replace both.

**9. 🟡 What is AOT compilation?**
Templates compiled to JS at build time: faster rendering, smaller bundles, template errors at build time, more secure. Default since v9 (Ivy).

**10. 🟡 What is Ivy?**
Angular's rendering engine/compiler (default v9): locality, tree-shakable runtime, smaller bundles, faster builds, enabling standalone and signals.

**11. 🟢 View encapsulation modes?**
`Emulated` (default, scoped attributes), `ShadowDom` (native shadow DOM), `None` (global styles).

**12. 🟢 What is content projection?**
Passing markup into a component with `<ng-content>`, multi-slot via `select`.

**13. 🟡 `ng-template` vs `ng-container` vs `ng-content`?**
`ng-template`: lazy template not rendered until instantiated. `ng-container`: grouping element that renders no DOM. `ng-content`: projection slot.

**14. 🟡 What is `@let`?**
Template-local variable declaration: `@let total = price() * qty();`.

**15. 🟡 How do you create a component dynamically?**
`ViewContainerRef.createComponent(Comp)` + `setInput`, or `NgComponentOutlet` in the template.

---

## B. Control Flow & Defer (03)

**16. 🟢 `@if`/`@for`/`@switch` vs `*ngIf`/`*ngFor`?**
Built-in syntax (v17): no imports, better type narrowing, faster, `@empty`, mandatory `track`. Structural directives deprecated in v20.

**17. 🟢 Why is `track` required in `@for`?**
Identifies items so DOM nodes are reused on changes — performance and preserved state.

**18. 🟡 What is `@defer`?**
Lazy-loads a template block's dependencies into a separate chunk on triggers (`idle`, `viewport`, `interaction`, `hover`, `timer`, `immediate`, `when`), with `@placeholder`, `@loading`, `@error`.

**19. 🔴 What are hydrate triggers?**
`@defer (hydrate on viewport)` etc. — with SSR the block is server-rendered and hydrated only when triggered (incremental hydration, default in v22).

---

## C. Signals (04)

**20. 🟢 What is a signal?**
A reactive value wrapper; reading tracks dependencies, writing notifies consumers. `signal`, `set`, `update`.

**21. 🟢 computed vs effect?**
`computed`: pure, lazy, memoized derived value. `effect`: side effects when dependencies change (sync to DOM/storage/logging).

**22. 🟡 What is `linkedSignal`?**
Writable signal derived from a source; resets when the source changes but can be set manually.

**23. 🟡 What does `untracked` do?**
Reads signals without creating dependencies.

**24. 🟡 Signals vs Observables?**
Signals: synchronous current value, automatic tracking, UI state. Observables: streams of events, operators, cancellation, async orchestration. Bridge with `toSignal`/`toObservable`.

**25. 🟡 What are `input()`, `model()`, `output()`?**
Signal input (read-only), two-way bindable writable input, and event emitter functions replacing `@Input`/`@Output`.

**26. 🟡 Why shouldn't effects set other signals?**
Leads to extra change-detection cycles, loops and hard-to-follow flows; use `computed`/`linkedSignal` for derived state.

**27. 🔴 How do signals enable zoneless?**
Signals notify Angular exactly which view to refresh, so Angular doesn't need zone.js to guess when something might have changed.

---

## D. Component Communication (05)

**28. 🟢 Parent → child?** Inputs. **Child → parent?** Outputs.

**29. 🟢 Sibling communication?**
Shared service with signals (or `BehaviorSubject`), or via common parent.

**30. 🟡 What is `viewChild`?**
Signal query to access a child component/element/directive from the parent class.

**31. 🟡 Why `BehaviorSubject` for shared state?**
Holds current value, emits it to new subscribers.

---

## E. Dependency Injection (06)

**32. 🟢 What is DI?**
A pattern where dependencies are provided from outside; Angular injectors create and supply them.

**33. 🟢 What does `providedIn: 'root'` do?**
App-wide, lazily created, tree-shakable singleton. `@Service()` in v22 is the shorthand.

**34. 🟡 Injector hierarchy?**
Element injectors (component/directive providers) up the tree, then environment injectors (route → root → platform) → NullInjector error.

**35. 🟡 Provider recipes?**
`useClass`, `useValue`, `useFactory`, `useExisting`, plus `multi: true`.

**36. 🟡 What is `InjectionToken`?**
A typed token for non-class dependencies (config, primitives, interfaces).

**37. 🟡 `inject()` vs constructor injection?**
`inject()` works in fields and functions, less boilerplate, better inheritance; requires injection context.

**38. 🟡 How to get a new service instance per component?**
Add it to the component's `providers`.

**39. 🔴 `providers` vs `viewProviders`?**
`viewProviders` are visible to the component's view only, not to projected content.

**40. 🔴 Resolution modifiers?**
`optional`, `self`, `skipSelf`, `host`.

**41. 🔴 Why can a "singleton" have two instances?**
Provided in a lazy route/module as well as root — the lazy environment injector creates its own.

---

## F. Change Detection (07)

**42. 🟢 What is change detection?**
Re-evaluating template bindings and updating the DOM when state may have changed.

**43. 🟡 What does zone.js do?**
Patches async APIs to notify Angular to run change detection after any async task.

**44. 🟡 When is an OnPush component checked?**
Input reference change, template/host event, signal read in template changes, `async` pipe emits, `markForCheck()`.

**45. 🟡 What changed in v22 regarding OnPush?**
OnPush is the default; old Default is `ChangeDetectionStrategy.Eager`.

**46. 🟡 What is zoneless?**
Angular without zone.js; CD runs only on explicit notifications (signals, events, markForCheck). Default for new apps since v21.

**47. 🟡 `markForCheck` vs `detectChanges`?**
Mark dirty for next cycle vs run CD synchronously now.

**48. 🔴 `ExpressionChangedAfterItHasBeenCheckedError`?**
Dev-mode error when a binding changes after being checked in the same cycle — unidirectional data-flow violation; fix data flow, use computed/signals.

**49. 🔴 What is `NgZone.runOutsideAngular`?**
Runs code without triggering CD (high-frequency events) in zone apps.

---

## G. RxJS (08)

**50. 🟢 Observable vs Promise?**
Lazy, multi-value, cancellable, operators vs eager, single value, not cancellable.

**51. 🟡 switchMap vs mergeMap vs concatMap vs exhaustMap?**
Cancel previous / parallel / queue / ignore while busy.

**52. 🟡 Subject types?**
`Subject`, `BehaviorSubject`, `ReplaySubject`, `AsyncSubject`.

**53. 🟡 Hot vs cold?**
Cold = producer per subscriber (HTTP); hot = shared producer (Subjects, events).

**54. 🟡 `forkJoin` vs `combineLatest`?**
Last values once all complete vs latest values whenever any emits.

**55. 🟡 How to avoid subscription leaks?**
`async` pipe, `toSignal`, `takeUntilDestroyed`, `takeUntil(destroy$)`.

**56. 🔴 Why put `catchError` inside `switchMap`?**
So an error doesn't kill the outer stream.

**57. 🔴 `shareReplay` pitfalls?**
Without `refCount: true`, the source stays subscribed forever (leak for infinite sources).

---

## H. HTTP (09)

**58. 🟢 How to make HTTP calls?**
`HttpClient` (provided with `provideHttpClient()`), returns cold observables.

**59. 🟡 What are interceptors?**
Middleware for requests/responses: auth headers, errors, loading, caching, retries. Functional with `withInterceptors`.

**60. 🟡 Why clone requests in interceptors?**
`HttpRequest` is immutable.

**61. 🟡 What is `httpResource`?**
Signal-based reactive HTTP reads with `value/isLoading/error/reload`; stable in v22.

**62. 🟡 What backend does HttpClient use in v22?**
Fetch by default; `withXhr()` to use XMLHttpRequest.

**63. 🔴 How do you handle token refresh?**
Interceptor catches 401, calls a shared (single-flight) refresh, retries the original request with the new token.

---

## I. Routing (10)

**64. 🟢 How does lazy loading work?**
`loadComponent`/`loadChildren` with dynamic `import()` creates separate chunks loaded on navigation.

**65. 🟡 Types of guards?**
`canActivate`, `canActivateChild`, `canDeactivate`, `canMatch`, `resolve`.

**66. 🟡 `canActivate` vs `canMatch`?**
`canMatch` decides whether the route matches (prevents lazy chunk download, falls through to next route).

**67. 🟡 What is a resolver?**
Fetches data before route activation.

**68. 🟡 Snapshot vs observable params?**
Snapshot doesn't update when the component is reused for a new param.

**69. 🟡 Preloading strategies?**
`NoPreloading`, `PreloadAllModules`, custom (selective, on idle).

**70. 🟡 What is `withComponentInputBinding`?**
Binds route params, query params and data to component inputs.

**71. 🟢 `routerLink` vs `href`?**
`routerLink` navigates client-side without full reload.

---

## J. Forms (11)

**72. 🟢 Template-driven vs reactive?**
Template-driven: `ngModel`, model in template, simple. Reactive: explicit typed control tree in class, testable, dynamic.

**73. 🟡 What are Signal Forms?**
v22-stable signal-based forms: model signal + `form()` + schema validators + `[formField]`.

**74. 🟡 `FormGroup` vs `FormArray`?**
Fixed named controls vs dynamic list.

**75. 🟡 `setValue` vs `patchValue`?**
Full structure required vs partial.

**76. 🟡 How to write a custom validator?**
Function `(control) => ValidationErrors | null`; group-level for cross-field.

**77. 🔴 What is `ControlValueAccessor`?**
Interface (`writeValue`, `registerOnChange`, `registerOnTouched`, `setDisabledState`) letting custom components work with forms; registered via `NG_VALUE_ACCESSOR`.

**78. 🔴 Async validators?**
Return Observable/Promise; run after sync validators; `PENDING` status; debounce them.

---

## K. Pipes & Directives (12)

**79. 🟢 What is a pipe?**
Template value transformer: `{{ date | date:'short' }}`.

**80. 🟡 Pure vs impure pipe?**
Pure re-runs on input reference change (memoized); impure every CD.

**81. 🟡 What does the async pipe do?**
Subscribes, returns latest value, marks for check, unsubscribes on destroy.

**82. 🟢 Types of directives?**
Components, attribute, structural.

**83. 🔴 What are host directives?**
Directive composition API: `hostDirectives` applies standalone directives to a component's host.

---

## L. State, Performance, SSR, Security, Testing (13–18)

**84. 🟡 How do you manage state?**
Local signals; signal service stores; NgRx Store / SignalStore for complex apps; resources for server data; URL for shareable state.

**85. 🟡 NgRx flow?**
Action → reducer → state → selector; effects for side effects.

**86. 🟡 How to optimize performance?**
Measure; lazy routes, `@defer`, lean bundles, `NgOptimizedImage`, SSR; OnPush + signals + zoneless, `track`, virtual scroll, pure pipes/computed, debounce.

**87. 🟡 Why avoid method calls in templates?**
They run every CD cycle.

**88. 🟡 What is `NgOptimizedImage`?**
Directive (`ngSrc`) enforcing dimensions, lazy loading, priority hints, srcset.

**89. 🟡 CSR vs SSR vs SSG?**
Browser render vs per-request server render vs build-time prerender; Angular picks per route with `RenderMode`.

**90. 🟡 What is hydration?**
Client reuses server DOM; incremental hydration hydrates `@defer` blocks on demand; event replay captures early events.

**91. 🟡 How to handle `window` with SSR?**
`afterNextRender`, `isPlatformBrowser`, inject `DOCUMENT`.

**92. 🟡 How does Angular prevent XSS?**
Escaping interpolation; context-aware sanitization; `bypassSecurityTrust*` only for trusted values.

**93. 🟡 CSRF support?**
HttpClient cookie-to-header (`XSRF-TOKEN` → `X-XSRF-TOKEN`).

**94. 🟡 How do you test a component?**
TestBed, `createComponent`, `setInput`, DOM interaction, `whenStable`, mock providers; Vitest default.

**95. 🟡 How do you test HTTP?**
`provideHttpClientTesting`, `HttpTestingController.expectOne`, `flush`, `verify`.

---

## M. Architecture & Modern Angular (19–21)

**96. 🟡 How do you structure a large app?**
Feature folders/libraries (data-access, feature, ui, util), core & shared, lazy routes, boundaries, smart/presentational split.

**97. 🟡 What is an NgModule? `forRoot`?**
Grouping of declarables & providers; `forRoot` provides singletons once, `forChild` for lazy modules.

**98. 🟡 What's new in Angular 22?**
Signal Forms & resources stable, OnPush default (`Eager` rename), `@Service()`, `injectAsync`, Fetch default, incremental hydration default, strict templates default, Aria GA, TS 6, yearly releases.

**99. 🔴 How would you migrate a legacy app?**
`ng update` step by step; esbuild builder; standalone, control-flow, signal input/output/query, inject migrations; adopt signals/OnPush; enable zoneless; Vitest.

**100. 🔴 Angular vs React?**
Angular: full framework, TypeScript-first, DI, opinionated structure, signals; React: UI library, JSX, ecosystem choices, hooks, virtual DOM. See `24-angular-vs-react.md`.

---

## Scenario Questions (senior) 🔴

1. **A list of 10,000 rows is slow.** → `track` by id, virtual scroll (CDK), OnPush/signals, pagination, avoid template methods, profile.
2. **Users double-submit a payment form.** → `exhaustMap`, disable while submitting, idempotency key on server.
3. **Search results show stale data out of order.** → `switchMap` (or `httpResource`) to cancel stale requests.
4. **The initial bundle is 3 MB.** → analyze, lazy routes, `@defer`, remove heavy libs, budgets.
5. **Memory grows when navigating.** → unsubscribe infinite streams, remove listeners/intervals, check root caches, DevTools heap snapshots.
6. **A child's update throws NG0100.** → move update earlier/into event, use computed/signals.
7. **Service state isn't shared between lazy features.** → provided in route providers twice; move to root/`@Service()`.
8. **SEO for product pages.** → SSR/SSG with render modes, meta/OG tags, JSON-LD, sitemap, incremental hydration for performance.
9. **Admin code is downloaded by normal users.** → `canMatch` guard on the lazy route.
10. **Migrating to zoneless.** → move state to signals, async pipe/toSignal, remove NgZone hooks, OnPush, test, then `provideZonelessChangeDetection()`.
