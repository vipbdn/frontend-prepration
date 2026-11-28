# 15 — SSR, Hydration & SEO 🔥🔥

> **Interview + Understanding Guide** — Roadmap Section 15
>
> Goal: Explain rendering strategies (CSR, SSR, SSG/prerender, hybrid), set up Angular SSR with per-route render modes, understand hydration (full, incremental — default in v22 — event replay, transfer cache), write platform-safe code, and handle SEO.

---

# 1. Rendering Strategies ⭐⭐⭐

| Strategy | HTML generated | Pros | Cons | Use for |
| --- | --- | --- | --- | --- |
| **CSR** (client-side) | In the browser by JS | Simple hosting, rich interactivity | Blank page until JS loads, weaker SEO, slow LCP on slow devices | Dashboards, apps behind login |
| **SSR** (server-side) | On the server **per request** | Fast first paint, SEO, fresh/personalized data | Server cost, TTFB, platform-safe code needed | Product pages, news, personalized pages |
| **SSG / Prerender** | At **build time** | Fastest, CDN-cacheable, cheap | Stale until rebuild, not per-user | Marketing, docs, blogs |
| **Hybrid** | Per route choice | Best of each | More config | Most real apps |

Angular supports all of them via **`@angular/ssr`** with route-level **render modes**.

---

# 2. Setup

```bash
ng new my-app --ssr            # or
ng add @angular/ssr
```

Generates `server.ts` (Express-based Node server), `app.config.server.ts`, `app.routes.server.ts`.

```ts
// app.config.ts
providers: [
  provideRouter(routes),
  provideClientHydration(withEventReplay()),   // hydration + replay early clicks
  provideHttpClient(),
]
```

```ts
// app.routes.server.ts — per-route render modes
import { RenderMode, ServerRoute } from "@angular/ssr";

export const serverRoutes: ServerRoute[] = [
  { path: "", renderMode: RenderMode.Prerender },                 // SSG
  { path: "about", renderMode: RenderMode.Prerender },
  {
    path: "products/:id",
    renderMode: RenderMode.Prerender,
    async getPrerenderParams() {                                   // which ids to prerender
      const ids = await inject(ProductApi).topIds();
      return ids.map((id) => ({ id }));
    },
  },
  { path: "account/**", renderMode: RenderMode.Client },          // CSR (behind login)
  { path: "**", renderMode: RenderMode.Server },                  // SSR per request
];
```

Server routes can also set `status` and `headers` (e.g. 404 pages, cache-control).

---

# 3. Hydration ⭐⭐⭐

**Hydration** = the client **reuses the server-rendered DOM** instead of destroying and re-creating it, attaching event listeners and app state.

```text
Without hydration (old "destructive" SSR): server HTML → JS boots → wipes DOM → re-renders → flicker, CLS, wasted work
With hydration: server HTML → JS boots → matches existing DOM nodes → attaches listeners → no flicker
```

## 3.1 Full hydration (v16+)

`provideClientHydration()`. Requirements: the server and client DOM must match — avoid direct DOM manipulation (`innerHTML`, `appendChild`) on server-rendered content; invalid HTML nesting (`<p><div>`) breaks it. Opt out a component with `ngSkipHydration` attribute.

## 3.2 Event replay (stable v19)

`withEventReplay()` — captures user events (clicks, inputs) that happen **before** hydration finishes and **replays** them afterwards, so early interactions aren't lost. Default in new apps.

## 3.3 Incremental hydration ⭐ (stable v20, **default in v22** 🆕)

Leave parts of the page **dehydrated** (static server HTML) and hydrate them **only when needed**, using `@defer` hydrate triggers:

```html
@defer (hydrate on viewport) {
  <app-product-reviews [id]="id()" />
}

@defer (hydrate on interaction) {
  <app-add-to-cart [product]="product()" />
}

@defer (hydrate never) {
  <app-static-footer />          <!-- pure content, never needs JS -->
}
```

Triggers: `hydrate on idle | viewport | interaction | hover | immediate | timer(ms)`, `hydrate when cond`, `hydrate never`.

- Server renders the **main content** of deferred blocks (not the placeholder).
- JS for that block is downloaded and executed only on trigger → **less JS, better INP/TBT**.
- 🆕 v22: on by default; `withIncrementalHydration()` deprecated; opt out via `withNoIncrementalHydration()`.

## 3.4 HTTP transfer cache

With `provideClientHydration()`, `HttpClient` GET/HEAD responses made on the server are **serialized into the HTML** and reused on the client → no duplicate requests during hydration. Configure with `withHttpTransferCacheOptions({ includePostRequests, filter, includeHeaders })`. Resources can use an `id` for SSR caching (v22).

## 3.5 Hydration errors (`NG0500`-series)

DOM mismatch, text mismatch, missing nodes — caused by direct DOM manipulation, browser-only content rendered differently, invalid HTML, or random/time-based values. Fix the mismatch or use `ngSkipHydration`.

---

# 4. Writing Platform-Safe Code ⭐⭐

On the server there's **no `window`, `document`, `localStorage`, `navigator`, `IntersectionObserver`**.

```ts
// ✅ Browser-only work after render
export class Chart {
  constructor() {
    afterNextRender(() => {
      // runs only in the browser, after the first render
      this.initChartLib(this.canvas().nativeElement);
    });
  }
  private readonly canvas = viewChild.required<ElementRef<HTMLCanvasElement>>("canvas");
  private initChartLib(_el: HTMLCanvasElement) {}
}

// ✅ Platform check
export class StorageService {
  private readonly isBrowser = isPlatformBrowser(inject(PLATFORM_ID));
  get(key: string) {
    return this.isBrowser ? localStorage.getItem(key) : null;
  }
}

// ✅ Inject DOCUMENT instead of global document
private readonly document = inject(DOCUMENT);
```

Also: `@defer` content is client-only in non-hydrate mode (placeholder rendered on the server); `PendingTasks` keeps the server waiting for async work; avoid long-lived `setInterval` on the server (app never becomes stable).

---

# 5. SEO ⭐⭐

```ts
import { Meta, Title } from "@angular/platform-browser";

export class ProductDetail {
  private readonly title = inject(Title);
  private readonly meta = inject(Meta);
  private readonly document = inject(DOCUMENT);
  readonly product = input.required<Product>();   // from resolver

  constructor() {
    effect(() => {
      const p = this.product();
      this.title.setTitle(`${p.name} | MyShop`);
      this.meta.updateTag({ name: "description", content: p.summary });
      this.meta.updateTag({ property: "og:title", content: p.name });
      this.meta.updateTag({ property: "og:image", content: p.imageUrl });
      this.meta.updateTag({ name: "twitter:card", content: "summary_large_image" });
    });
  }
}
```

SEO checklist:
- **SSR/SSG** so crawlers and social previews get full HTML
- Unique `<title>` (route `title` or custom `TitleStrategy`) and meta description per page
- Open Graph / Twitter tags
- **Canonical** link (`<link rel="canonical">` via `DOCUMENT`)
- **Structured data** (JSON-LD `<script type="application/ld+json">`)
- `sitemap.xml`, `robots.txt`
- Proper HTTP status for 404s (server route `status: 404`)
- Semantic HTML, `alt` text, real `<a href>` links (`routerLink` renders them)
- Performance (Core Web Vitals are a ranking signal)
- `lang` attribute, i18n `hreflang`

---

# 6. Interview Answers

**CSR vs SSR vs SSG?**

> "CSR renders in the browser — simple to host but the user sees little until JavaScript loads, and SEO and LCP suffer. SSR renders HTML on the server per request — fast first paint, good SEO, fresh or personalized data, at the cost of server load. SSG prerenders at build time — fastest and CDN-cacheable but static until rebuilt. Angular lets me choose per route with server routes and `RenderMode.Server`, `Client` or `Prerender`."

**What is hydration? Incremental hydration?**

> "Hydration means the client reuses the server-rendered DOM, attaching listeners and state instead of re-rendering, which avoids flicker and layout shift. Event replay records interactions before hydration and replays them. Incremental hydration — default in Angular 22 — leaves `@defer` blocks as static server HTML and only downloads and hydrates them on a trigger like viewport or interaction, cutting the JavaScript executed upfront."

**How do you handle browser-only APIs with SSR?**

> "Use `afterNextRender`/`afterEveryRender` for DOM work, which only run in the browser; check `isPlatformBrowser(PLATFORM_ID)` for things like localStorage; inject `DOCUMENT` instead of using globals; and defer browser-only widgets with `@defer`. Also avoid direct DOM manipulation on server-rendered content, which causes hydration mismatches."

**How do you improve SEO in Angular?**

> "Render with SSR or prerendering so crawlers get real content, set per-page titles and meta/Open Graph tags with the `Title` and `Meta` services, add canonical links and JSON-LD structured data, generate a sitemap, return correct status codes, use semantic HTML with real links, and keep Core Web Vitals healthy."

---

# 7. Key Takeaways

- Hybrid rendering per route with `RenderMode`.
- Hydration reuses DOM; event replay; incremental hydration default in v22 via `@defer (hydrate on …)`.
- Transfer cache prevents duplicate HTTP on hydration.
- Platform-safe code: `afterNextRender`, `isPlatformBrowser`, `DOCUMENT`.
- SEO = SSR + meta + structured data + performance.

## Readiness Checklist

- [ ] Configure server routes with all three render modes
- [ ] Add incremental hydration triggers to a product page
- [ ] Make a localStorage service SSR-safe
- [ ] Set dynamic title/meta/OG tags
