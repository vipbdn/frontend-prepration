# 🅰️ Angular Interview Roadmap — Latest Angular (v22)

> A structured Angular preparation track for a **Frontend Developer (3–5 YOE)**, written for **Angular 22** (released June 2026) — the signal-first, standalone, zoneless, OnPush-by-default era — with notes on older patterns you'll still meet in existing codebases and interviews.
>
> **Prerequisites:** the JavaScript and TypeScript notes (closures, `this`, promises, event loop, generics, decorators basics).

---

## 📊 Progress Tracker

| # | Section | File | Status |
| --- | --- | --- | --- |
| 1 | Angular Fundamentals & Architecture | `01-angular-fundamentals.md` | ⬜ Not Started |
| 2 | Components, Templates & Data Binding | `02-components-templates-binding.md` | ⬜ Not Started |
| 3 | Control Flow, Templates & `@defer` | `03-control-flow-and-defer.md` | ⬜ Not Started |
| 4 | Signals | `04-signals.md` | ⬜ Not Started |
| 5 | Component Communication | `05-component-communication.md` | ⬜ Not Started |
| 6 | Dependency Injection | `06-dependency-injection.md` | ⬜ Not Started |
| 7 | Change Detection & Zoneless | `07-change-detection-zoneless.md` | ⬜ Not Started |
| 8 | RxJS in Angular | `08-rxjs.md` | ⬜ Not Started |
| 9 | HTTP, Interceptors & Resources | `09-http-and-resources.md` | ⬜ Not Started |
| 10 | Routing | `10-routing.md` | ⬜ Not Started |
| 11 | Forms (Template, Reactive, Signal Forms) | `11-forms.md` | ⬜ Not Started |
| 12 | Pipes & Directives | `12-pipes-and-directives.md` | ⬜ Not Started |
| 13 | State Management (Signals, NgRx, SignalStore) | `13-state-management.md` | ⬜ Not Started |
| 14 | Performance Optimization | `14-performance.md` | ⬜ Not Started |
| 15 | SSR, Hydration & SEO | `15-ssr-hydration-seo.md` | ⬜ Not Started |
| 16 | Testing | `16-testing.md` | ⬜ Not Started |
| 17 | Angular Material, CDK & Aria | `17-material-cdk-aria.md` | ⬜ Not Started |
| 18 | Security | `18-security.md` | ⬜ Not Started |
| 19 | Architecture & Best Practices | `19-architecture-best-practices.md` | ⬜ Not Started |
| 20 | NgModules & Legacy Angular | `20-ngmodules-and-legacy.md` | ⬜ Not Started |
| 21 | What's New: Version History v14 → v22 | `21-version-history-whats-new.md` | ⬜ Not Started |
| 22 | Rapid-Fire Interview Q&A | `22-interview-questions.md` | ⬜ Not Started |
| 23 | Machine Coding in Angular | `23-machine-coding.md` | ⬜ Not Started |
| 24 | Angular vs React | `24-angular-vs-react.md` | ⬜ Not Started |

### Legend

⬜ Not Started · 🔄 In Progress · ✅ Completed · ⭐ High Priority · 🔥 Very High Priority

---

# Topic Checklist

## 1. Fundamentals
- [ ] What Angular is; framework vs library; Angular vs AngularJS
- [ ] ⭐ Architecture: components, templates, services, DI, router, signals
- [ ] Angular CLI & project structure (`ng new`, `ng generate`, `ng serve`, `ng build`)
- [ ] ⭐ Standalone bootstrapping: `bootstrapApplication`, `app.config.ts`, providers
- [ ] Build system: esbuild/Vite application builder

## 2. Components & Templates 🔥
- [ ] ⭐ `@Component` metadata (selector, template, styles, imports, changeDetection, host)
- [ ] ⭐ Data binding: interpolation, property, attribute, class/style, event, two-way
- [ ] Template reference variables, `@let`
- [ ] ⭐ Lifecycle hooks (`ngOnInit`, `ngOnChanges`, `ngOnDestroy`, `ngAfterViewInit`…) + `afterNextRender`/`afterEveryRender`
- [ ] View encapsulation; `:host`; content projection (`ng-content`, multi-slot)

## 3. Control Flow & Defer 🔥
- [ ] ⭐ `@if`, `@for` (with `track`), `@switch`
- [ ] ⭐ `@defer` (triggers, prefetch, placeholder/loading/error)
- [ ] `ng-template`, `ng-container`, `NgTemplateOutlet`
- [ ] Legacy `*ngIf`/`*ngFor`/`*ngSwitch` (deprecated)

## 4. Signals 🔥🔥🔥
- [ ] ⭐ `signal`, `computed`, `effect`, `untracked`
- [ ] ⭐ `linkedSignal`
- [ ] ⭐ Signal inputs `input()`, `input.required()`, `model()`, `output()`
- [ ] Signal queries `viewChild()`, `viewChildren()`, `contentChild()`, `contentChildren()`
- [ ] ⭐ RxJS interop: `toSignal`, `toObservable`, `outputFromObservable`
- [ ] `debounced()` (experimental, v22)

## 5. Component Communication 🔥
- [ ] ⭐ Parent→child (inputs), child→parent (outputs), two-way (`model`)
- [ ] ⭐ Shared services with signals
- [ ] ViewChild / template refs; content projection; router state

## 6. Dependency Injection 🔥🔥
- [ ] ⭐ Injector hierarchy (environment vs element injectors)
- [ ] ⭐ `@Service()` (v22) / `@Injectable({ providedIn: 'root' })`
- [ ] ⭐ `inject()` vs constructor injection; injection context
- [ ] ⭐ Providers: `useClass`, `useValue`, `useFactory`, `useExisting`, `multi`
- [ ] ⭐ `InjectionToken`
- [ ] Resolution modifiers: `optional`, `self`, `skipSelf`, `host`
- [ ] `injectAsync()` (v22), `DestroyRef`, `runInInjectionContext`

## 7. Change Detection 🔥🔥🔥
- [ ] ⭐ How change detection works (tree, top-down, dirty checking)
- [ ] ⭐ zone.js vs zoneless (default for new apps since v21)
- [ ] ⭐ OnPush (default in v22) vs Eager (formerly Default)
- [ ] `markForCheck`, `detectChanges`, signals marking views dirty
- [ ] ⭐ `ExpressionChangedAfterItHasBeenCheckedError`

## 8. RxJS 🔥🔥
- [ ] ⭐ Observable vs Promise vs Signal
- [ ] ⭐ Higher-order mapping: `switchMap`, `mergeMap`, `concatMap`, `exhaustMap`
- [ ] ⭐ `debounceTime`, `distinctUntilChanged`, `combineLatest`, `forkJoin`, `catchError`, `retry`
- [ ] ⭐ Subjects (`Subject`, `BehaviorSubject`, `ReplaySubject`)
- [ ] Hot vs cold; `shareReplay`; unsubscribing (`takeUntilDestroyed`, `async` pipe)

## 9. HTTP & Resources 🔥
- [ ] ⭐ `provideHttpClient`, Fetch default (v22), typed requests
- [ ] ⭐ Functional interceptors (auth, errors, loading, caching)
- [ ] ⭐ `resource`, `rxResource`, `httpResource` (stable v22), `chain()`
- [ ] Error handling, retry, `HttpTestingController`

## 10. Routing 🔥
- [ ] ⭐ `provideRouter`, routes, nested routes, `RouterOutlet`, `RouterLink`
- [ ] ⭐ Lazy loading (`loadComponent`, `loadChildren`) & preloading
- [ ] ⭐ Functional guards & resolvers (`canActivate`, `canMatch`, `canDeactivate`)
- [ ] ⭐ Route params via `withComponentInputBinding`
- [ ] Router events, titles, view transitions

## 11. Forms 🔥🔥
- [ ] Template-driven forms (`ngModel`)
- [ ] ⭐ Typed Reactive Forms (`FormGroup`, `FormControl`, `FormArray`, `FormBuilder`)
- [ ] ⭐ Validators: built-in, custom, async, cross-field
- [ ] ⭐ Signal Forms (stable v22): `form()`, schema validators, `FormField`, `submit()`
- [ ] `ControlValueAccessor` / `FormValueControl`

## 12. Pipes & Directives
- [ ] Built-in pipes; pure vs impure; custom pipes
- [ ] Attribute & structural directives; host bindings; `hostDirectives`

## 13–24. Advanced & Interview
- [ ] ⭐ State: signal services, NgRx Store, NgRx SignalStore
- [ ] 🔥 Performance: OnPush, signals, `@defer`, lazy loading, `NgOptimizedImage`, track, virtual scroll
- [ ] ⭐ SSR, hydration (incremental by default in v22), render modes, SEO with `Title`/`Meta`
- [ ] ⭐ Testing: Vitest (default), TestBed, harnesses, HttpTestingController
- [ ] Material, CDK, Angular Aria
- [ ] ⭐ Security: sanitization, XSS, XSRF, CSP
- [ ] ⭐ Architecture: style guide, feature folders, Nx, micro-frontends
- [ ] NgModules & legacy patterns (still common in interviews)
- [ ] 🔥 100 rapid-fire Q&A · machine coding · Angular vs React

---

# 🎯 Learning Order

| Phase | Files | Goal |
| --- | --- | --- |
| 1 — Foundations | 01, 02, 03 | Components, templates, control flow |
| 2 — Reactivity | 04, 05, 07 | Signals, communication, change detection |
| 3 — Services | 06, 08, 09 | DI, RxJS, HTTP & resources |
| 4 — App building | 10, 11, 12 | Routing, forms, pipes & directives |
| 5 — Scale | 13, 14, 15 | State, performance, SSR |
| 6 — Professional | 16, 17, 18, 19 | Testing, UI libraries, security, architecture |
| 7 — Legacy & versions | 20, 21 | NgModules, what changed per version |
| 8 — Interview | 22, 23, 24 | Q&A, machine coding, comparisons |

---

# 🏆 Must-Master List (most asked)

1. [ ] Component lifecycle hooks (and their order)
2. [ ] Data binding types; `@Input`/`@Output` vs `input()`/`output()`/`model()`
3. [ ] Signals: `signal`/`computed`/`effect`/`linkedSignal` and when to use each
4. [ ] Change detection, OnPush, zoneless, `ExpressionChanged…` error
5. [ ] Dependency injection hierarchy & providers
6. [ ] RxJS `switchMap` vs `mergeMap` vs `concatMap` vs `exhaustMap`
7. [ ] Subjects & `BehaviorSubject`
8. [ ] Unsubscribing & memory leaks
9. [ ] Observables vs Promises vs Signals
10. [ ] HTTP interceptors
11. [ ] Lazy loading & preloading strategies
12. [ ] Route guards & resolvers
13. [ ] Reactive vs template-driven vs Signal Forms; custom validators
14. [ ] Pure vs impure pipes
15. [ ] Structural vs attribute directives
16. [ ] Content projection & `ng-template`
17. [ ] Standalone components vs NgModules
18. [ ] `@defer` and control flow
19. [ ] SSR & hydration
20. [ ] Performance optimization techniques
21. [ ] NgRx architecture (actions, reducers, effects, selectors) & SignalStore
22. [ ] Testing components & services
23. [ ] Security & sanitization
24. [ ] What's new in recent versions (v17 → v22)

> **Interview tip:** Angular interviews often mix **modern** questions (signals, standalone, zoneless) with **classic** ones (NgModules, `@Input`, zone.js, lifecycle). Be fluent in both and explain *why* Angular moved forward.
