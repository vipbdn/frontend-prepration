# 🅰️ Angular Interview Notes — Angular 22

Interview-focused Markdown notes for a **frontend developer with 3–5 years of experience**, written for **Angular 22** (June 2026): standalone, signals, zoneless, OnPush by default, Signal Forms and resources stable. The notes also cover the legacy patterns (NgModules, decorators, zone.js, `*ngIf`) you'll still find in real codebases and interviews.

Each file follows the same structure, so they work for **learning**, **revision** and **last-minute interview prep**:

```text
Concept → How it works → Code → Pitfalls → Output questions → Interview answers (say-it-aloud) → Key takeaways → Readiness checklist
```

Markers: ⭐ high priority · 🔥 very frequently asked · ⚠️ pitfall · 🆕 new in v21/v22 · 🧪 output question

---

## 📚 Files

| # | File | What's inside |
| --- | --- | --- |
| 00 | [00-angular-roadmap.md](00-angular-roadmap.md) | Progress tracker, topic checklist, learning order |
| 01 | [01-angular-fundamentals.md](01-angular-fundamentals.md) | Architecture, CLI, project structure, bootstrapping, AOT/Ivy |
| 02 | [02-components-templates-binding.md](02-components-templates-binding.md) | Components, bindings, lifecycle, encapsulation, projection, dynamic components |
| 03 | [03-control-flow-and-defer.md](03-control-flow-and-defer.md) | `@if`/`@for`/`@switch`, `@defer`, ng-template/ng-container |
| 04 | [04-signals.md](04-signals.md) | signal, computed, effect, linkedSignal, signal inputs/queries, RxJS interop |
| 05 | [05-component-communication.md](05-component-communication.md) | Inputs/outputs/model, viewChild, shared services, router |
| 06 | [06-dependency-injection.md](06-dependency-injection.md) | Injectors, providers, InjectionToken, `@Service()`, `injectAsync` |
| 07 | [07-change-detection-zoneless.md](07-change-detection-zoneless.md) | zone.js, OnPush vs Eager, signals, zoneless, NG0100 |
| 08 | [08-rxjs.md](08-rxjs.md) | Observables, Subjects, the four flattening operators, unsubscribing, typeahead |
| 09 | [09-http-and-resources.md](09-http-and-resources.md) | HttpClient, interceptors, httpResource/resource/rxResource |
| 10 | [10-routing.md](10-routing.md) | Routes, lazy loading, preloading, guards, resolvers, input binding |
| 11 | [11-forms.md](11-forms.md) | Template-driven, reactive, Signal Forms, validators, CVA |
| 12 | [12-pipes-and-directives.md](12-pipes-and-directives.md) | Pure/impure pipes, async pipe, custom directives, hostDirectives |
| 13 | [13-state-management.md](13-state-management.md) | Signal stores, NgRx Store, NgRx SignalStore |
| 14 | [14-performance.md](14-performance.md) | Load and runtime optimization, NgOptimizedImage, virtual scroll |
| 15 | [15-ssr-hydration-seo.md](15-ssr-hydration-seo.md) | Render modes, hydration, incremental hydration, SEO |
| 16 | [16-testing.md](16-testing.md) | Vitest, TestBed, harnesses, HTTP/router testing, Playwright |
| 17 | [17-material-cdk-aria.md](17-material-cdk-aria.md) | Material, CDK, Angular Aria, accessibility |
| 18 | [18-security.md](18-security.md) | XSS and sanitization, XSRF, CSP, token storage |
| 19 | [19-architecture-best-practices.md](19-architecture-best-practices.md) | Style guide, folder structure, smart/presentational, Nx, micro-frontends |
| 20 | [20-ngmodules-and-legacy.md](20-ngmodules-and-legacy.md) | NgModules, forRoot, legacy → modern mapping, migrations |
| 21 | [21-version-history-whats-new.md](21-version-history-whats-new.md) | v14 → v22 timeline, Angular 22 changes, upgrading |
| 22 | [22-interview-questions.md](22-interview-questions.md) | 100 rapid-fire Q&A + senior scenarios |
| 23 | [23-machine-coding.md](23-machine-coding.md) | 10 machine-coding solutions (todo, autocomplete, modal, tabs…) |
| 24 | [24-angular-vs-react.md](24-angular-vs-react.md) | Comparison and concept mapping |

---

## 🗓️ Study Plan (3 weeks)

| Week | Days | Files | Goal |
| --- | --- | --- | --- |
| 1 — Core | 1–2 | 01, 02, 03 | Build components with modern templates |
| | 3–4 | 04, 05 | Think in signals |
| | 5–6 | 06, 07 | Explain DI and change detection deeply (most-asked senior topics) |
| | 7 | Revise 01–07 + Q&A sections A–F in 22 | |
| 2 — App building | 8–9 | 08, 09 | RxJS operators and HTTP/resources |
| | 10–11 | 10, 11 | Routing and forms |
| | 12 | 12, 13 | Pipes, directives, state |
| | 13–14 | 23 (build 5 solutions from scratch) | Machine coding speed |
| 3 — Senior topics | 15–16 | 14, 15 | Performance and SSR |
| | 17 | 16, 18 | Testing and security |
| | 18 | 17, 19, 20 | Ecosystem, architecture, legacy |
| | 19 | 21, 24 | What's new, comparison |
| | 20–21 | 22 + mock interviews | Rapid revision |

**Short on time (3 days)?** 04 → 07 → 06 → 08 → 10 → 11 → 14 → 21 → 22.

---

## 🔁 Revision Method

1. **First pass:** read a file and type out every code block yourself.
2. **Active recall:** cover the "Interview Answers" and answer aloud in about 60 seconds, then compare.
3. **Output questions:** predict before opening the `<details>` answers.
4. **Checklist:** tick the readiness checklist only when you can do each item without notes.
5. **Spaced review:** revisit after 1, 3 and 7 days using file 22.

---

## ⚠️ Version Note

The code targets **Angular 22** APIs. A few are flagged **experimental** (for example `debounced()`), and some APIs differ in older versions (for example `@Service()` is v22+, and Signal Forms were experimental in v21). Each file points out legacy equivalents. Check [angular.dev](https://angular.dev) for the latest details before an interview.
