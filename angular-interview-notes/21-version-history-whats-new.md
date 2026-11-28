# 21 — Version History & What's New 🔥

> **Interview + Understanding Guide** — Roadmap Section 21
>
> Goal: Answer "what's new in Angular?" and "how has Angular evolved?" — the key features from v14 to **v22 (June 2026)**, the release/support policy, and how to upgrade.

---

# 1. Release Policy

- Until v21: **major every 6 months** (May & November).
- 🆕 From v22: **one major per year** (v22 June 2026, v23 expected June 2027), with minors in between.
- Each major: **active support** then **LTS** — check angular.dev/reference/releases for current dates.
- Upgrade guide: **update.angular.dev**; run `ng update` one major at a time.
- Deprecated APIs are removed after at least two majors (with migrations when possible).

---

# 2. The "Angular Renaissance" Timeline ⭐⭐⭐

| Version | Date | Headline features |
| --- | --- | --- |
| **v14** | Jun 2022 | Standalone components (preview), **typed reactive forms**, `inject()` in more places, page `title` on routes |
| **v15** | Nov 2022 | **Standalone stable**, directive composition (`hostDirectives`), `NgOptimizedImage` stable, functional guards/interceptors, better stack traces |
| **v16** | May 2023 | **Signals (preview)**, `takeUntilDestroyed`/`DestroyRef`, required inputs, route params → inputs, non-destructive hydration (preview), esbuild builder (preview) |
| **v17** | Nov 2023 | **Built-in control flow** (`@if/@for/@switch`), **`@defer`**, new branding & **angular.dev**, Vite/esbuild default builder, SSR simplified (`@angular/ssr`), hydration stable, signals stable (core) |
| **v18** | May 2024 | **Zoneless (experimental)**, Material 3 stable, `@let` (preview), event replay (preview), route `redirectTo` functions, unified form `events` |
| **v19** | Nov 2024 | **Standalone by default**, `linkedSignal`, `resource`/`rxResource` (experimental), incremental hydration (preview), route-level render modes, HMR for styles/templates, `@let` stable, signal inputs/outputs/queries stable |
| **v20** | May 2025 | **Signals APIs stable** (`effect`, `linkedSignal`, `toSignal`…), **zoneless (dev preview → stable in 20.2)**, incremental hydration stable, `httpResource` (experimental), **`*ngIf`/`*ngFor`/`*ngSwitch` deprecated**, new **style guide** (no suffixes), template exponentiation/`in`/tagged templates, native `animate.enter/leave` (20.2) |
| **v21** | Nov 2025 | **Zoneless by default** for new apps, **Signal Forms (experimental)**, **Vitest default** test runner, **Angular Aria** (preview), Angular **MCP server** / AI tooling, `@defer`/Regex in templates improvements |
| **v22** | Jun 2026 | See below ⬇️ |

---

# 3. Angular 22 — What's New ⭐⭐⭐ 🆕

### Signals & data
- **Signal Forms stable** — `form()`, `[formField]`, schema validators, `minDate`/`maxDate`, `debounce(field, 'blur')`, `validateAsync`/`validateHttp` with debounce, `reloadValidation()`, `getError()`; `FormValueControl` works with legacy forms.
- **`resource`, `rxResource`, `httpResource` stable**; `chain()` for dependent resources; SSR caching via `id`.
- `debounced()` signal utility (experimental).

### Change detection
- **OnPush is the default** strategy; old Default renamed **`ChangeDetectionStrategy.Eager`** (migration adds `Eager` to existing components).

### Dependency injection
- **`@Service()`** decorator (tree-shakable root singleton, `inject()`-only; `autoProvided: false` to opt out). CLI generates it.
- **`injectAsync()`** for lazy-loaded services with optional prefetch.

### HTTP
- **Fetch backend by default**; `withFetch()` deprecated, `withXhr()` to opt out; `reportUploadProgress`/`reportDownloadProgress` replace `reportProgress`.

### Router
- `canMatch` guards get a required 3rd parameter (current snapshot).
- `paramsInheritanceStrategy` defaults to **`'always'`**.
- `withComponentInputBinding()` options; `RouterLink` `browserUrl` input.

### Templates & compiler
- **`strictTemplates` on by default**.
- Comments inside element tags; `@switch` exhaustiveness (`@default never`), multiple cases; spread and arrow functions in expressions; optional chaining returns `undefined` like TS; `@defer (on idle(500ms))` timeout.

### SSR
- **Incremental hydration default**; `withIncrementalHydration()` deprecated; `withNoIncrementalHydration()` to opt out.

### Testing & tooling
- `TestBed.getLastFixture()`; Vitest `fakeAsync` support via `zone.js/plugins/vitest-patch`; Karma → Vitest migration schematic.
- **TypeScript 6 required**; Node 20 dropped.
- **Webpack builders deprecated** (use esbuild `application` builder).
- **Angular Aria GA**; experimental WebMCP support and AI agent skills.

---

# 4. Upgrading ⭐

```bash
ng version
ng update                                   # shows available updates
ng update @angular/core@22 @angular/cli@22  # one major at a time
ng update @angular/material@22
```

Checklist:
1. Read update.angular.dev for your from/to versions.
2. Update Node/TypeScript to supported versions (v22: TS 6, Node 22+).
3. Commit before each step; run migrations; run tests/build.
4. Fix deprecation warnings before they become removals.
5. Update third-party libs (NgRx, Material, etc.) to compatible versions.

---

# 5. Interview Answers

**What's new in the latest Angular?**

> "Angular 22, released June 2026, stabilizes the signal story: Signal Forms and the resource APIs including `httpResource` are stable. OnPush is now the default change detection strategy, with the old default renamed Eager. There's a new `@Service()` decorator and `injectAsync` for lazy-loaded services, HttpClient uses Fetch by default, incremental hydration is on by default for SSR, strict templates are enabled by default, and Angular Aria is GA. It requires TypeScript 6, deprecates webpack builders, and Angular moves to yearly major releases."

**How has Angular changed in recent years?**

> "It's become much simpler and more reactive. Standalone components removed the need for NgModules; signals brought fine-grained reactivity and enabled zoneless change detection, now the default; built-in control flow and `@defer` replaced structural directives and added declarative lazy loading; the CLI moved to esbuild and Vite; SSR got non-destructive and incremental hydration; and forms, inputs, outputs and queries all have signal-based APIs. Testing moved to Vitest."

---

# 6. Key Takeaways

- v14–v16: standalone, typed forms, signals preview.
- v17: control flow, `@defer`, esbuild, angular.dev.
- v19–v21: standalone default, signals stable, zoneless default, Vitest, Signal Forms experimental.
- v22: Signal Forms & resources stable, OnPush default, `@Service`, Fetch default, incremental hydration default, yearly releases.

## Readiness Checklist

- [ ] Name 3 headline features each for v16, v17, v20, v21, v22
- [ ] Explain the v22 change detection rename
- [ ] Describe a safe upgrade process
