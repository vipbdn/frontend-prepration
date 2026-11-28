# 14 — Performance Optimization 🔥🔥🔥

> **Interview + Understanding Guide** — Roadmap Section 14
>
> Goal: Diagnose and fix Angular performance problems — load time (bundle, lazy loading, `@defer`, images, SSR) and runtime (change detection, lists, signals, zone) — and answer "how would you optimize a slow Angular app?" confidently.

---

# 1. Measure First ⭐

| Tool | What it shows |
| --- | --- |
| **Angular DevTools** (Profiler, Injector tree, Signal graph) | CD cycles, time per component, what triggered CD |
| Chrome **Performance** panel | Long tasks, scripting, layout thrashing |
| **Lighthouse** / PageSpeed / CrUX | Core Web Vitals (LCP, INP, CLS), opportunities |
| `ng build --stats-json` + **esbuild bundle analyzer** / `source-map-explorer` | What's in the bundle |
| Bundle **budgets** in `angular.json` | Fail builds when bundles grow |

Core Web Vitals targets: **LCP < 2.5s**, **INP < 200ms**, **CLS < 0.1**.

---

# 2. Load-Time Performance ⭐⭐⭐

## 2.1 Lazy load routes

```ts
{ path: "admin", loadChildren: () => import("./admin/admin.routes").then((m) => m.ADMIN_ROUTES) }
```

+ preloading strategy (selective / on idle), `canMatch` to avoid downloading unauthorized code.

## 2.2 `@defer` — lazy-load parts of a template

```html
@defer (on viewport; prefetch on idle) {
  <app-reviews [productId]="id()" />
} @placeholder (minimum 300ms) {
  <div class="skeleton" style="height: 400px"></div>    <!-- reserve space → no CLS -->
} @loading (after 150ms) {
  <app-spinner />
}
```

Defer charts, comments, maps, rich editors, below-the-fold sections, modals.

## 2.3 Bundle hygiene

- **Production build** (`ng build`): AOT, minification, tree-shaking, dead-code elimination (esbuild/Vite builder; webpack builders deprecated in v22)
- Avoid importing whole libraries (`import _ from "lodash"` → `lodash-es` per-function or native)
- Replace heavy deps (moment → `Intl`/date-fns)
- Standalone components → better tree-shaking than big shared NgModules
- `injectAsync()` (v22) to code-split heavy services
- Zoneless removes zone.js (~30KB+)
- Check `angular.json` budgets

## 2.4 Images: `NgOptimizedImage` ⭐

```ts
import { NgOptimizedImage } from "@angular/common";

@Component({
  imports: [NgOptimizedImage],
  template: `
    <img ngSrc="/hero.jpg" width="1200" height="600" priority alt="Hero" />           <!-- LCP image -->
    <img ngSrc="product.jpg" width="300" height="300" alt="…" />                       <!-- lazy by default -->
    <img ngSrc="banner.jpg" fill sizes="100vw" placeholder alt="…" />                   <!-- responsive, blur placeholder -->
  `,
})
export class Gallery {}
```

Enforces width/height (prevents CLS), lazy-loads non-priority images, `fetchpriority=high` + preload hints for `priority`, generates `srcset` with image loaders (Cloudinary, Imgix, etc.), warns about oversized images.

## 2.5 SSR / SSG + hydration

Server rendering or prerendering improves FCP/LCP and SEO; incremental hydration (default in v22) hydrates sections on demand → less JS executed upfront. See `15-ssr-hydration-seo.md`.

## 2.6 Other

- Fonts: `font-display: swap`, inline critical fonts (Angular CLI inlines Google Fonts CSS), preconnect
- Critical CSS inlining (on by default)
- HTTP caching/CDN, compression (Brotli), HTTP/2+
- Service worker (`@angular/pwa`) for caching assets

---

# 3. Runtime Performance ⭐⭐⭐

## 3.1 Change detection

- **OnPush** (default v22) + immutable data
- **Signals** for state → only consuming views re-render
- **Zoneless** → no CD for unrelated async events
- No heavy computation in templates: **pure pipes** or **`computed`** instead of method calls/getters

```html
{{ calculateTotal(items) }}   <!-- ❌ runs every CD -->
{{ total() }}                 <!-- ✅ computed, memoized -->
```

## 3.2 Lists

```html
@for (item of items(); track item.id) { <app-row [item]="item" /> }
```

- `track` by a stable id → DOM nodes reused, not recreated (`track $index` only for static lists)
- **Virtual scrolling** for long lists (CDK):

```html
<cdk-virtual-scroll-viewport itemSize="56" class="viewport">
  <div *cdkVirtualFor="let user of users; trackBy: trackById" class="row">{{ user.name }}</div>
</cdk-virtual-scroll-viewport>
```

- Pagination / infinite scroll
- Avoid deep component nesting per row if not needed

## 3.3 High-frequency events

```ts
// zone.js apps: don't trigger CD on every mousemove
private readonly zone = inject(NgZone);
constructor() {
  this.zone.runOutsideAngular(() => {
    fromEvent<MouseEvent>(document, "mousemove").pipe(throttleTime(50), takeUntilDestroyed()).subscribe((e) => {
      if (this.shouldUpdate(e)) this.zone.run(() => this.position.set({ x: e.x, y: e.y }));
    });
  });
}
```

- Debounce inputs (`debounceTime`, Signal Forms `debounce`), throttle scroll/resize
- Use `IntersectionObserver`/`ResizeObserver` instead of scroll listeners
- `afterRenderEffect` phases to avoid layout thrashing (batch reads then writes)

## 3.4 Heavy computation

- Web Workers (`ng generate web-worker`)
- Memoize with `computed`
- Break work up (`scheduler.yield`, `requestIdleCallback`)

## 3.5 Memory leaks

- Unsubscribe infinite streams (`takeUntilDestroyed`, `async` pipe, `toSignal`)
- Remove manual DOM listeners/intervals (`DestroyRef.onDestroy`)
- Destroy dynamically created components
- Beware large caches in root services

## 3.6 HTTP

- Cancel stale requests (`switchMap`, resources do it automatically)
- Cache (`shareReplay`, transfer cache for SSR)
- Batch / paginate; request only needed fields

---

# 4. Checklist: "How would you optimize a slow Angular app?" ⭐⭐⭐

```text
1. Measure: Lighthouse/CrUX (load), DevTools Profiler (runtime), bundle analyzer.
2. Load:
   - Lazy routes + preloading, @defer heavy/below-fold UI
   - Trim bundle: remove heavy deps, tree-shakable imports, budgets
   - NgOptimizedImage (priority LCP image, sizes, lazy)
   - SSR/SSG + incremental hydration for content pages
   - Caching/CDN/compression
3. Runtime:
   - OnPush + signals (+ zoneless)
   - track in @for, virtual scroll for long lists
   - pure pipes/computed instead of template methods
   - debounce/throttle, runOutsideAngular, web workers
4. Fix leaks: subscriptions, listeners, timers.
5. Re-measure, set budgets & monitor in CI/RUM.
```

---

# 5. Interview Answers

**How do you optimize Angular performance?**

> "I start by measuring — Lighthouse and real-user Web Vitals for loading, Angular DevTools' profiler for change detection, and a bundle analyzer. For load time: lazy-loaded routes with preloading, `@defer` for heavy or below-the-fold components, trimming dependencies, `NgOptimizedImage` with a priority LCP image, and SSR or prerendering with incremental hydration for content pages. For runtime: OnPush and signals — which are now the default with zoneless — `track` in loops, virtual scrolling for long lists, pure pipes or computeds instead of template method calls, debouncing input, and moving heavy work to web workers. Then I fix subscription leaks and add bundle budgets to keep it from regressing."

**Why is calling a method in a template bad?**

> "Template expressions are re-evaluated on every change detection cycle for that component, so a method runs constantly — expensive if it computes anything. A pure pipe or a `computed` signal memoizes the result and only recalculates when inputs change."

**What does `track` in `@for` do?**

> "It tells Angular how to identify items. When the list changes, Angular reuses DOM nodes and component instances for items with the same key instead of destroying and recreating them — preserving state and avoiding expensive DOM work. Track by a stable unique id."

**What is `@defer` and how does it help?**

> "A template block that splits its content into a separate lazy chunk, loaded on a trigger such as viewport, idle, interaction or hover, with placeholder and loading states. It reduces the initial bundle and JavaScript work, improving LCP and INP; with SSR it also enables incremental hydration."

---

# 6. Key Takeaways

- Measure → fix → re-measure; budgets in CI.
- Load: lazy routes, `@defer`, lean bundles, `NgOptimizedImage`, SSR/hydration.
- Runtime: OnPush + signals + zoneless, `track`, virtual scroll, memoization, debounce.
- Clean up subscriptions and listeners.

## Readiness Checklist

- [ ] Profile an app with Angular DevTools
- [ ] Add `@defer` + skeleton without CLS
- [ ] Use `NgOptimizedImage` with a priority image
- [ ] Implement CDK virtual scroll
- [ ] Give the 5-step optimization answer from memory
