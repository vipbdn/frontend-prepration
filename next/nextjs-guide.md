# ▲ The Complete Next.js Guide — Learning, Revision & Interview Prep

> A practical guide to **learn Next.js**, **revise quickly**, and **answer interview questions in depth** — with special focus on **rendering paradigms**, **caching**, **optimization** and **SEO**.
>
> Written for the **App Router (Next.js 15–16)**, with notes on what changed across versions and a Pages Router reference for legacy interviews. Prerequisites: React fundamentals (see the React notes) and basic TypeScript.

---

## 📖 How to Use This Guide

| If you want to… | Go to |
| --- | --- |
| Learn Next.js from scratch | Sections 1 → 9 in order |
| Understand CSR / SSR / SSG / ISR / Streaming / PPR | [§5 Rendering Paradigms](#5-rendering-paradigms-) |
| Understand caching (the most confusing topic) | [§7 Caching & Revalidation](#7-caching--revalidation-) |
| Make an app fast | [§11 Optimization](#11-optimization-) |
| Make an app rank on Google | [§12 SEO](#12-seo-) |
| Fix an error right now | [§20 Common Errors & Fixes](#20-common-errors--fixes-) |
| Prepare for an interview | [§24 Interview Questions (detailed)](#24-interview-questions--detailed-answers-) |

**Legend:** ⭐ very important / frequently asked · ⚠️ pitfall · 💡 tip · 🆕 recent version change

---

## 📑 Table of Contents

1. [What is Next.js?](#1-what-is-nextjs)
2. [Setup & Project Structure](#2-setup--project-structure)
3. [Routing (App Router)](#3-routing-app-router-)
4. [Server & Client Components](#4-server--client-components-)
5. [Rendering Paradigms](#5-rendering-paradigms-)
6. [Data Fetching](#6-data-fetching-)
7. [Caching & Revalidation](#7-caching--revalidation-)
8. [Mutations: Server Actions & Route Handlers](#8-mutations-server-actions--route-handlers-)
9. [Proxy (Middleware)](#9-proxy-middleware)
10. [Authentication & Authorization](#10-authentication--authorization-)
11. [Optimization](#11-optimization-)
12. [SEO](#12-seo-)
13. [Styling](#13-styling)
14. [Configuration & Environment Variables](#14-configuration--environment-variables)
15. [Error Handling & Observability](#15-error-handling--observability)
16. [Internationalization (i18n)](#16-internationalization-i18n)
17. [Testing](#17-testing)
18. [Deployment](#18-deployment)
19. [Pages Router Reference (Legacy)](#19-pages-router-reference-legacy)
20. [Common Errors & Fixes](#20-common-errors--fixes-)
21. [Best Practices](#21-best-practices-)
22. [Real-World Recipes](#22-real-world-recipes)
23. [Cheat Sheet](#23-cheat-sheet-)
24. [Interview Questions — Detailed Answers](#24-interview-questions--detailed-answers-)
25. [Practice Projects](#25-practice-projects)

---

# 1. What is Next.js?

**Next.js** is a **full-stack React framework** by Vercel. React gives you components; Next.js adds everything needed to ship a production app:

| Feature | What Next.js provides |
| --- | --- |
| Routing | File-system based routing with nested layouts |
| Rendering | Server Components, SSR, SSG, ISR, streaming, Partial Prerendering — chosen **per route/component** |
| Data | Async Server Components, caching, revalidation |
| Mutations | Server Actions, Route Handlers (API endpoints) |
| Performance | Automatic code splitting, prefetching, `next/image`, `next/font`, `next/script` |
| SEO | Server-rendered HTML, Metadata API, sitemaps, robots, OG images |
| Edge logic | Proxy/Middleware for redirects, rewrites, auth checks |
| Tooling | Turbopack/webpack bundling, TypeScript, ESLint integration, fast refresh |
| Deployment | Vercel, Node server, Docker, static export |

## 1.1 Why not plain React (Vite SPA)?

| React SPA (Vite) | Next.js |
| --- | --- |
| Client-side rendering only → empty HTML until JS runs | Server-rendered HTML → fast first paint, SEO-friendly |
| You choose a router, data fetching, SSR setup | Integrated, conventions included |
| Every component ships JS | Server Components ship **zero JS** |
| Simple static hosting | Needs a Node/edge runtime for dynamic features (or static export) |
| Great for dashboards behind login, embedded apps | Great for public sites, e-commerce, content, SaaS, full-stack apps |

## 1.2 App Router vs Pages Router ⭐

| | Pages Router (`pages/`) | App Router (`app/`) |
| --- | --- | --- |
| Introduced | 2016 | Next 13 (2022), stable 13.4 |
| Components | Client components (SSR'd + hydrated) | **Server Components by default** |
| Data fetching | `getServerSideProps`, `getStaticProps`, `getStaticPaths` | `async` components, `fetch`/ORM + caching, `generateStaticParams` |
| Layouts | `_app`, manual per-page layouts | Nested persistent `layout.tsx` |
| Loading/error UI | Manual | `loading.tsx`, `error.tsx`, `not-found.tsx` |
| Streaming | Limited | Built-in with Suspense |
| Mutations | API routes | Server Actions + Route Handlers |
| Status | Supported (maintenance) | **Recommended for new apps** |

Both can coexist in one project during migration.

## 1.3 Version timeline (what changed when) 🆕

| Version | Highlights |
| --- | --- |
| **13** (2022) | App Router (beta → stable in 13.4), Server Components, `next/font`, Turbopack alpha |
| **14** (2023) | Server Actions stable, Partial Prerendering preview |
| **15** (Oct 2024) | React 19; **async request APIs** (`params`, `searchParams`, `cookies()`, `headers()` return Promises); **`fetch` and GET Route Handlers no longer cached by default**; client router cache no longer reuses page segments by default; `after()`, `<Form>`, `next.config.ts`, Turbopack dev stable, `instrumentation.ts` stable |
| **16** (Oct 2025) | **Turbopack default** for dev and build; **Cache Components** (`"use cache"`, `cacheLife`, `cacheTag`) unify caching + PPR; **`middleware.ts` → `proxy.ts`**; `updateTag()`/`refresh()`, `revalidateTag(tag, profile)`; React 19.2; React Compiler support stable; sync access to request APIs removed; `next lint` removed; AMP removed |

💡 In interviews, explain **concepts** (static vs dynamic, caching layers, revalidation) and mention that **defaults changed** in 15/16 — interviewers appreciate version awareness.

---

# 2. Setup & Project Structure

```bash
npx create-next-app@latest my-app   # prompts: TypeScript, ESLint/Biome, Tailwind, src/, App Router, alias
cd my-app
npm run dev       # dev server (Turbopack) → http://localhost:3000
npm run build     # production build (shows which routes are static ○ / dynamic ƒ)
npm run start     # run the production build
```

### Typical structure

```text
my-app/
├── app/                      # App Router (routes live here)
│   ├── layout.tsx            # root layout (required: <html>, <body>)
│   ├── page.tsx              # "/"
│   ├── globals.css
│   ├── (marketing)/          # route group (not in URL)
│   │   └── pricing/page.tsx  # "/pricing"
│   ├── blog/
│   │   ├── page.tsx          # "/blog"
│   │   └── [slug]/page.tsx   # "/blog/:slug"
│   ├── dashboard/
│   │   ├── layout.tsx
│   │   ├── loading.tsx
│   │   ├── error.tsx
│   │   └── page.tsx
│   ├── api/health/route.ts   # Route Handler → GET /api/health
│   ├── sitemap.ts            # /sitemap.xml
│   └── robots.ts             # /robots.txt
├── components/               # shared UI (or inside app/_components)
├── lib/                      # db client, auth, data access layer, utils
├── public/                   # static files served at "/" (favicon, images)
├── proxy.ts                  # (Next 16; middleware.ts in ≤15) runs before routes
├── next.config.ts
├── tsconfig.json
└── package.json
```

💡 **Colocation**: you can put components, tests and styles inside `app/` — only `page.tsx` and `route.ts` make a segment publicly routable. Prefix folders with `_` (private folders, e.g. `_components`) to exclude them from routing explicitly.

---

# 3. Routing (App Router) ⭐

## 3.1 File conventions ⭐

| File | Purpose |
| --- | --- |
| `page.tsx` | UI for a route — makes the segment publicly accessible |
| `layout.tsx` | Shared UI wrapping child segments; **persists** across navigation (state preserved, not re-rendered) |
| `template.tsx` | Like layout but **remounts** on every navigation (enter animations, per-page effects) |
| `loading.tsx` | Loading UI — automatically wraps the page in `<Suspense>` |
| `error.tsx` | Error boundary for the segment (must be a Client Component) |
| `global-error.tsx` | Error boundary for the root layout |
| `not-found.tsx` | UI for `notFound()` and unmatched URLs |
| `route.ts` | API endpoint (Route Handler) — can't coexist with `page.tsx` in the same segment |
| `default.tsx` | Fallback for parallel route slots |
| `opengraph-image.tsx`, `icon.tsx`, `sitemap.ts`, `robots.ts`, `manifest.ts` | Metadata files (§12) |

### Component hierarchy of a segment

```text
<Layout>
  <Template>
    <ErrorBoundary fallback={<Error />}>
      <Suspense fallback={<Loading />}>
        <ErrorBoundary fallback={<NotFound />}>
          <Page />
        </ErrorBoundary>
      </Suspense>
    </ErrorBoundary>
  </Template>
</Layout>
```

⚠️ `error.tsx` in a segment does **not** catch errors in that segment's own `layout.tsx` — they bubble to the parent's error boundary (or `global-error.tsx`).

## 3.2 Pages and layouts

```tsx
// app/layout.tsx — root layout (Server Component)
import type { Metadata } from "next";
import "./globals.css";

export const metadata: Metadata = {
  title: { default: "Acme", template: "%s | Acme" },
  description: "Acme store",
};

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="en">
      <body>
        <Header />
        <main>{children}</main>
        <Footer />
      </body>
    </html>
  );
}
```

```tsx
// app/dashboard/layout.tsx — nested layout
export default function DashboardLayout({ children }: { children: React.ReactNode }) {
  return (
    <div className="grid grid-cols-[240px_1fr]">
      <Sidebar />          {/* stays mounted while navigating between dashboard pages */}
      <section>{children}</section>
    </div>
  );
}
```

## 3.3 Dynamic routes ⭐

| Folder | Matches | `params` |
| --- | --- | --- |
| `[slug]` | `/blog/hello` | `{ slug: "hello" }` |
| `[...slug]` (catch-all) | `/shop/a/b/c` | `{ slug: ["a", "b", "c"] }` |
| `[[...slug]]` (optional catch-all) | `/docs` and `/docs/a/b` | `{ slug: undefined }` or array |

```tsx
// app/blog/[slug]/page.tsx (Next 15+: params and searchParams are Promises 🆕)
import { notFound } from "next/navigation";

type Props = {
  params: Promise<{ slug: string }>;
  searchParams: Promise<{ [key: string]: string | string[] | undefined }>;
};

export default async function BlogPost({ params, searchParams }: Props) {
  const { slug } = await params;
  const { ref } = await searchParams;
  const post = await getPost(slug);
  if (!post) notFound();               // renders not-found.tsx with 404 status
  return <article><h1>{post.title}</h1></article>;
}
```

In Client Components, read params with hooks: `useParams()`, `useSearchParams()`, `usePathname()` — or unwrap the promise with React's `use(params)`.

## 3.4 Route groups, private folders

```text
app/(marketing)/about/page.tsx   → /about      (group organizes code / different layouts)
app/(shop)/layout.tsx            → layout only for shop routes
app/(auth)/login/page.tsx        → /login
app/_lib/utils.ts                → not routable
```

Multiple root layouts are possible with groups (`app/(site)/layout.tsx`, `app/(admin)/layout.tsx`) — navigating between them triggers a full page load.

## 3.5 Parallel & intercepting routes (advanced)

**Parallel routes** render multiple pages in the same layout simultaneously via **slots** (`@name`):

```text
app/dashboard/
├── layout.tsx         // receives { children, analytics, team }
├── page.tsx
├── @analytics/page.tsx
└── @team/page.tsx
```

Each slot can have its own loading/error states — great for dashboards and conditional UI (e.g. show `@admin` only to admins).

**Intercepting routes** load a route **within the current layout** (e.g. as a modal) while keeping the real URL shareable — the Instagram photo modal pattern:

```text
app/feed/page.tsx
app/feed/@modal/(.)photo/[id]/page.tsx   ← clicking a photo in the feed opens a modal
app/photo/[id]/page.tsx                  ← direct visit / refresh shows the full page
```

Conventions: `(.)` same level, `(..)` one level up, `(...)` from root.

## 3.6 Navigation ⭐

```tsx
import Link from "next/link";

<Link href="/blog/hello">Read</Link>
<Link href={{ pathname: "/search", query: { q: "shoes" } }}>Search</Link>
<Link href="/dashboard" prefetch={false}>Dashboard</Link>
<Link href="/terms" replace scroll={false}>Terms</Link>
```

- `<Link>` renders a real `<a href>` (crawlable, accessible) and performs **client-side navigation** (no full reload).
- **Prefetching** (production): links in the viewport are prefetched — static routes fully, dynamic routes up to the nearest `loading.tsx` boundary. 🆕 Next 16 adds layout deduplication and incremental prefetching to reduce prefetch requests.

```tsx
"use client";
import { useRouter, usePathname, useSearchParams } from "next/navigation"; // ⚠️ not "next/router" (Pages Router)

export function Filters() {
  const router = useRouter();
  const pathname = usePathname();
  const searchParams = useSearchParams();

  function setCategory(category: string) {
    const params = new URLSearchParams(searchParams);
    params.set("category", category);
    router.push(`${pathname}?${params}`);   // also: replace, back, forward, refresh, prefetch
  }
  return <button onClick={() => setCategory("shoes")}>Shoes</button>;
}
```

Server-side navigation helpers (Server Components, Server Actions, Route Handlers):

```ts
import { redirect, permanentRedirect, notFound } from "next/navigation";
redirect("/login");            // 307 (303 in Server Actions)
permanentRedirect("/new-url"); // 308
notFound();                    // 404 + not-found.tsx
```

## 3.7 Route segment config

```ts
// in page.tsx / layout.tsx / route.ts
export const dynamic = "auto";        // "force-dynamic" | "force-static" | "error"
export const revalidate = 3600;       // ISR interval in seconds (false = cache forever, 0 = dynamic)
export const dynamicParams = true;    // false → 404 for params not returned by generateStaticParams
export const runtime = "nodejs";      // or "edge"
export const preferredRegion = "auto";
export const maxDuration = 30;        // serverless function timeout (platform-dependent)
```

🆕 With **Cache Components** enabled (Next 16), `dynamic`/`revalidate` segment configs are replaced by `"use cache"` + `cacheLife` (§7.4).

---

# 4. Server & Client Components ⭐

## 4.1 The default: Server Components

Every component in `app/` is a **React Server Component (RSC)** unless marked otherwise.

```tsx
// app/products/page.tsx — Server Component
import { db } from "@/lib/db";

export default async function ProductsPage() {
  const products = await db.product.findMany();         // runs on the server: direct DB access
  return (
    <ul>
      {products.map((p) => (
        <li key={p.id}>
          {p.name} <AddToCartButton productId={p.id} />   {/* client island */}
        </li>
      ))}
    </ul>
  );
}
```

```tsx
// components/AddToCartButton.tsx — Client Component
"use client";
import { useState } from "react";

export function AddToCartButton({ productId }: { productId: string }) {
  const [added, setAdded] = useState(false);
  return <button onClick={() => setAdded(true)}>{added ? "Added ✓" : "Add to cart"}</button>;
}
```

## 4.2 Comparison ⭐⭐⭐

| | Server Component | Client Component (`"use client"`) |
| --- | --- | --- |
| Where it runs | Server only (build time or request time) | Pre-rendered on the server (SSR) **and** hydrated/run in the browser |
| JS sent to browser | **None** | Yes |
| `async`/`await` in component | ✅ | ❌ (use `use()`, React Query, SWR) |
| DB, filesystem, secrets, server SDKs | ✅ | ❌ |
| `useState`, `useEffect`, `useReducer`, context | ❌ | ✅ |
| Event handlers (`onClick`) | ❌ | ✅ |
| Browser APIs (`window`, `localStorage`) | ❌ | ✅ (in effects/handlers) |
| Heavy libraries (markdown, syntax highlighting, date libs) | ✅ stay on server (0 KB) | Add to bundle |

### When to use which

- **Server**: fetching data, accessing backend resources, keeping secrets, large dependencies, static content — the default.
- **Client**: interactivity (state, events), effects, browser APIs, custom hooks using state, context providers, class components.

## 4.3 The `"use client"` boundary ⭐

- `"use client"` at the top of a file marks it — **and everything it imports** — as part of the client bundle.
- You don't need it in every client file — only at the **entry point** of the client subtree.
- **Push client boundaries to the leaves**: make the small interactive piece a Client Component, not the whole page.

```text
Page (server)
├── Header (server)
│   └── SearchBox (client)  ← only this ships JS
├── ProductGrid (server)
│   └── AddToCartButton (client)
└── Footer (server)
```

## 4.4 Composition rules ⭐

1. **Server → Client**: Server Components can import and render Client Components.
2. **Client ✗→ Server**: Client Components **cannot import** Server Components…
3. …but they can **receive** Server Components as `children` or other props (slots):

```tsx
// ✅ Server page passes server-rendered content into a client wrapper
<ClientTabs>
  <ServerRenderedReviews />    {/* rendered on the server, passed as a prop */}
</ClientTabs>
```

4. **Props from Server to Client must be serializable**: primitives, plain objects, arrays, `Date`, `Map`, `Set`, Promises, JSX — **not** functions (except Server Actions), class instances, or symbols.

```tsx
// ❌ Error: Functions cannot be passed directly to Client Components
<ClientButton onClick={() => console.log("hi")} />
// ✅ define the handler inside the Client Component, or pass a Server Action
```

## 4.5 Context providers

Context requires client components. Create a `providers.tsx` and wrap `children` in the root layout:

```tsx
// app/providers.tsx
"use client";
import { QueryClient, QueryClientProvider } from "@tanstack/react-query";
import { ThemeProvider } from "next-themes";
import { useState } from "react";

export function Providers({ children }: { children: React.ReactNode }) {
  const [queryClient] = useState(() => new QueryClient());
  return (
    <QueryClientProvider client={queryClient}>
      <ThemeProvider attribute="class">{children}</ThemeProvider>
    </QueryClientProvider>
  );
}
```

Because `children` are passed in from the (server) layout, wrapping them in a client provider **doesn't** turn the whole app into client components.

## 4.6 Keeping server code on the server

```ts
// lib/db.ts
import "server-only";   // build error if imported into a Client Component
```

`client-only` does the opposite. Also note: only `NEXT_PUBLIC_*` env vars are available in the browser.

## 4.7 Third-party components without `"use client"`

Some libraries use hooks but don't declare `"use client"`. Wrap them:

```tsx
// components/carousel.tsx
"use client";
export { Carousel } from "some-carousel-lib";
```

---

# 5. Rendering Paradigms ⭐⭐⭐

The most asked Next.js topic. Understand **where and when HTML is generated**.

## 5.1 Overview

| Paradigm | HTML generated | Data freshness | Speed | SEO | Use for |
| --- | --- | --- | --- | --- | --- |
| **CSR** — Client-Side Rendering | In the browser after JS loads | Real-time | Slow first paint | Weak | Dashboards, highly interactive tools behind login |
| **SSR** — Server-Side Rendering (dynamic) | On the server **per request** | Always fresh | Good (depends on server/data time) | ✅ | Personalized pages, search results, carts, auth-dependent content |
| **SSG** — Static Site Generation | At **build time** | Stale until rebuild | ⚡ Fastest (CDN) | ✅ | Marketing, docs, blogs |
| **ISR** — Incremental Static Regeneration | At build, then **regenerated in background** after revalidation | Fresh within interval / on demand | ⚡ CDN speed | ✅ | Product pages, blogs with many pages, news |
| **Streaming SSR** | Server, sent **progressively** in chunks | Fresh | Fast TTFB/FCP | ✅ | Pages with slow data parts |
| **PPR** — Partial Prerendering / Cache Components | **Static shell** at build + **dynamic holes** streamed per request | Mixed | ⚡ + fresh | ✅ | Most real pages: static layout + personalized parts |

```text
CSR:    request → empty HTML + JS → JS runs → fetch data → render           (content late)
SSR:    request → server fetches data → HTML → browser shows → hydrate       (content early)
SSG:    build → HTML on CDN → request → instant HTML → hydrate
ISR:    build → HTML on CDN → after N seconds, next request gets stale page AND triggers regeneration
Stream: request → shell + fallbacks immediately → slow parts streamed in as they resolve
PPR:    request → static shell from CDN instantly → dynamic parts streamed into holes
```

## 5.2 Static vs Dynamic rendering in the App Router ⭐⭐⭐

In the App Router you don't pick SSR/SSG with special functions — Next.js decides **per route** based on what your code uses:

### A route is **static** (rendered at build time, cached, served from CDN) by default…

…unless it uses **dynamic APIs** or uncached data, which makes it **dynamic** (rendered per request):

| Makes a route dynamic |
| --- |
| `cookies()`, `headers()`, `draftMode()` |
| `searchParams` prop in a page |
| `connection()` (opt into dynamic explicitly) |
| `fetch(url, { cache: "no-store" })` (and, in Next 15+, uncached fetches during render by default in some configurations) |
| `export const dynamic = "force-dynamic"` |
| `unstable_noStore()` (older) |

The build output shows each route: **○ Static**, **● SSG** (static with `generateStaticParams`), **ƒ Dynamic**, and with PPR **◐ Partial Prerender**.

### SSG with dynamic params: `generateStaticParams`

```tsx
// app/blog/[slug]/page.tsx
export async function generateStaticParams() {
  const posts = await getAllPosts();
  return posts.map((post) => ({ slug: post.slug }));   // these pages are built at build time
}

export const dynamicParams = true;  // other slugs: rendered on demand, then cached (default)
// false → unknown slugs return 404
```

Equivalent to Pages Router `getStaticPaths` + `getStaticProps`.

### ISR

```tsx
export const revalidate = 3600;   // regenerate at most once per hour (segment config)

// or per fetch
const res = await fetch("https://api.example.com/products", { next: { revalidate: 3600, tags: ["products"] } });
```

**How ISR works (stale-while-revalidate):**

```text
t=0     build → page generated & cached
t<1h    requests → cached page (fast)
t=1h+   first request → still gets the STALE page, but triggers background regeneration
        regeneration succeeds → cache updated → subsequent requests get the fresh page
        regeneration fails → keeps serving the last good page
On-demand: revalidatePath("/blog") / revalidateTag("posts") after a CMS webhook → regenerate immediately
```

### Dynamic (SSR)

```tsx
import { cookies } from "next/headers";

export default async function AccountPage() {
  const session = (await cookies()).get("session")?.value;   // dynamic API → rendered per request
  const user = await getUserFromSession(session);
  return <h1>Welcome back, {user.name}</h1>;
}
```

### CSR in Next.js

Still possible and appropriate for highly interactive, user-specific widgets:

```tsx
"use client";
import useSWR from "swr";

export function Notifications() {
  const { data, isLoading } = useSWR("/api/notifications", (url) => fetch(url).then((r) => r.json()));
  if (isLoading) return <Skeleton />;
  return <List items={data} />;
}
```

Or `next/dynamic` with `{ ssr: false }` for browser-only components (maps, charts using `window`).

## 5.3 Streaming ⭐

With Suspense, Next.js sends the page **progressively**:

```tsx
// app/product/[id]/page.tsx
import { Suspense } from "react";

export default async function ProductPage({ params }: { params: Promise<{ id: string }> }) {
  const { id } = await params;
  const product = await getProduct(id);              // fast, needed for the shell
  return (
    <>
      <ProductDetails product={product} />
      <Suspense fallback={<ReviewsSkeleton />}>
        <Reviews productId={id} />                    {/* slow: streamed when ready */}
      </Suspense>
      <Suspense fallback={<RecommendationsSkeleton />}>
        <Recommendations productId={id} />
      </Suspense>
    </>
  );
}

async function Reviews({ productId }: { productId: string }) {
  const reviews = await getReviews(productId);      // 2s API — doesn't block the rest of the page
  return <ReviewList reviews={reviews} />;
}
```

- `loading.tsx` = automatic Suspense boundary around the whole page.
- Benefits: better **TTFB**, **FCP**, perceived performance; slow data doesn't block fast data; **selective hydration** (React hydrates interactive parts as they arrive and prioritizes what the user interacts with).
- SEO: streamed content is still in the HTML response (the response just arrives in chunks), so crawlers that wait for the full response see it.

## 5.4 Partial Prerendering (PPR) & Cache Components 🆕⭐

**Problem:** a route was all-static or all-dynamic. One `cookies()` call (e.g. a cart badge) made the whole product page dynamic.

**PPR:** prerender a **static shell** at build time (layout, nav, product info) and leave **holes** (Suspense boundaries) for dynamic content that is **streamed in at request time** — in the **same HTTP response**.

```tsx
// Next 16: enable in next.config.ts → cacheComponents: true
import { Suspense } from "react";

export default function ProductPage() {
  return (
    <>
      <StaticProductInfo />                    {/* prerendered into the shell */}
      <Suspense fallback={<CartSkeleton />}>
        <CartBadge />                          {/* reads cookies → dynamic hole, streamed */}
      </Suspense>
    </>
  );
}
```

With **Cache Components** (Next 16), everything is **dynamic by default** and you **opt into caching** with `"use cache"` — the static shell is made of everything cacheable; anything reading request data must be inside `<Suspense>` (the build tells you if it isn't). See §7.4.

## 5.5 Hydration

**Hydration** = React in the browser attaching event handlers and state to the server-rendered HTML of Client Components. The first client render must **match** the server HTML.

Common hydration mismatch causes: `Date.now()`/`new Date().toLocaleString()`, `Math.random()`, `typeof window` branches, `localStorage` reads during render, invalid HTML nesting (`<p><div>`), browser extensions. Fixes: move browser-only logic into `useEffect`, `useId` for ids, `dynamic(..., { ssr: false })`, `suppressHydrationWarning` for unavoidable timestamps. (Details in §20.)

## 5.6 Runtimes: Node.js vs Edge

| | Node.js runtime (default) | Edge runtime |
| --- | --- | --- |
| APIs | Full Node.js (fs, native modules, most npm packages) | Web APIs subset (fetch, Request/Response, crypto) |
| Cold start | Slower | Very fast |
| Location | Region(s) | Globally distributed, close to users |
| Limits | Generous | Code size & API limits |
| Use | Most pages, DB access, heavy logic | Lightweight personalization, redirects, geolocation |

🆕 Next 16's `proxy.ts` runs on the **Node.js** runtime (the old `middleware.ts` ran on Edge).

## 5.7 Choosing a rendering strategy ⭐

```text
Is the content the same for every user?
├─ Yes → Does it change?
│        ├─ Rarely / on deploy → SSG (static)
│        └─ Periodically / via CMS → ISR (time-based or on-demand revalidation)
└─ No (personalized / per request) →
         ├─ SEO needed or fast first paint? → Dynamic SSR with streaming (or PPR: static shell + dynamic holes)
         └─ Behind login & highly interactive? → Server shell + client components (CSR for live widgets)
```

| Page | Strategy |
| --- | --- |
| Landing page, docs | SSG |
| Blog with 10k posts | SSG for popular + `dynamicParams` on-demand + ISR/tags |
| Product page | ISR / Cache Components (static info) + dynamic stock/price/cart holes |
| Search results (`?q=`) | Dynamic SSR + streaming |
| User dashboard | Dynamic SSR shell + client components / React Query |
| Real-time chat | Client components + WebSocket |

---

# 6. Data Fetching ⭐

## 6.1 Fetch in Server Components

```tsx
export default async function Page() {
  const res = await fetch("https://api.example.com/posts", { next: { revalidate: 60 } });
  if (!res.ok) throw new Error("Failed to fetch posts");   // caught by error.tsx
  const posts: Post[] = await res.json();
  return <PostList posts={posts} />;
}
```

Or query your database / ORM directly (Prisma, Drizzle) — no API layer needed for reads:

```tsx
const orders = await db.order.findMany({ where: { userId }, take: 20 });
```

⚠️ Don't call your **own** Route Handlers (`fetch("/api/...")`) from Server Components — it adds a network hop; call the data function directly.

## 6.2 Sequential vs parallel ⭐ (avoid waterfalls)

```tsx
// ❌ Waterfall: 1s + 1s + 1s
const user = await getUser(id);
const orders = await getOrders(id);
const recs = await getRecommendations(id);

// ✅ Parallel: ~1s
const [user, orders, recs] = await Promise.all([getUser(id), getOrders(id), getRecommendations(id)]);

// ✅ Even better: stream independent parts with Suspense (each component fetches its own data)
```

Sequential is correct only when one request depends on another (`getOrders(user.id)`).

## 6.3 Request memoization & `React.cache` ⭐

During a single render pass, identical `fetch` calls (same URL & options) are **deduplicated** automatically. For non-fetch data (ORM calls), wrap with React's `cache`:

```ts
// lib/data.ts
import { cache } from "react";
import "server-only";

export const getUser = cache(async (id: string) => {
  return db.user.findUnique({ where: { id } });
});
```

Now `layout.tsx`, `page.tsx` and `generateMetadata` can all call `getUser(id)` — **one** DB query per request. (Scope: a single request; not shared across users/requests.)

## 6.4 Passing promises to Client Components (streaming data)

```tsx
// Server Component: start fetching, don't await
export default function Page() {
  const commentsPromise = getComments();
  return (
    <Suspense fallback={<CommentsSkeleton />}>
      <Comments commentsPromise={commentsPromise} />
    </Suspense>
  );
}
```

```tsx
// Client Component
"use client";
import { use } from "react";

export function Comments({ commentsPromise }: { commentsPromise: Promise<Comment[]> }) {
  const comments = use(commentsPromise);   // suspends until resolved
  return comments.map((c) => <p key={c.id}>{c.text}</p>);
}
```

## 6.5 Client-side fetching

Use TanStack Query / SWR in Client Components for: frequently changing data, user-triggered fetches, infinite scroll, polling, optimistic UI. Combine with server prefetching + `HydrationBoundary` for instant first render.

## 6.6 Where to fetch — decision table

| Data | Fetch in |
| --- | --- |
| Needed for first render, SEO-relevant | Server Component |
| Public, rarely changing | Server Component with caching (`"use cache"`/revalidate) |
| User-specific, needed for first paint | Server Component (dynamic) |
| Changes every few seconds, interactive filters, infinite lists | Client (React Query/SWR) |
| Triggered by a user action (search-as-you-type) | Client |

---

# 7. Caching & Revalidation ⭐⭐⭐

The most confusing — and frequently asked — Next.js topic. Learn the **layers**, then the **APIs**.

## 7.1 The caching layers

| Layer | What | Where | Duration | Purpose |
| --- | --- | --- | --- | --- |
| **Request Memoization** | Return values of identical `fetch`/`cache()` calls | Server, in memory | One request (render pass) | Dedupe within a render tree |
| **Data Cache** | `fetch` results (and `"use cache"` results) | Server, persistent | Across requests & deployments until revalidated | Avoid refetching data |
| **Full Route Cache** | Rendered HTML + RSC payload of static routes | Server / CDN | Until revalidated or redeployed | Serve static routes without rendering |
| **Router Cache** | RSC payloads of visited/prefetched segments | Browser memory | Session / time-based | Instant back/forward & navigation |

```text
Browser ──(Router Cache)──▶ Server ──(Full Route Cache)──▶ Render ──(Request Memoization)──▶ (Data Cache) ──▶ Data source
```

## 7.2 Default behaviour by version 🆕 ⭐

| | Next 13–14 | Next 15 | Next 16 with Cache Components |
| --- | --- | --- | --- |
| `fetch` | **Cached by default** (`force-cache`) | **Not cached** by default — opt in with `cache: "force-cache"` or `next.revalidate` | Dynamic by default; cache with `"use cache"` |
| GET Route Handlers | Cached (static) by default | Not cached by default | Same model as pages |
| Client Router Cache for pages | Reused for 30s (dynamic) / 5min (static) | `staleTime` 0 for page segments (layouts & `loading` still reused) | Same as 15 |
| Static vs dynamic | Inferred from dynamic API usage | Same | Explicit: cached = `"use cache"`, request data inside `<Suspense>` |

Why the change? Implicit caching surprised developers (stale data, hard-to-debug behaviour). The direction is **explicit, opt-in caching**.

## 7.3 Caching with `fetch` and segment config (Next 14/15 model)

```ts
fetch(url, { cache: "force-cache" });                   // cache indefinitely (until revalidated)
fetch(url, { cache: "no-store" });                      // never cache (dynamic)
fetch(url, { next: { revalidate: 60 } });               // time-based revalidation (ISR)
fetch(url, { next: { tags: ["products"] } });           // tag for on-demand revalidation

export const revalidate = 60;                           // route-level default
export const dynamic = "force-static";                  // force static
```

Caching non-fetch data (ORM) in this model: `unstable_cache(fn, keyParts, { revalidate, tags })` — superseded by `"use cache"`.

## 7.4 Cache Components: `"use cache"` (Next 16) 🆕⭐

```ts
// next.config.ts
import type { NextConfig } from "next";
const nextConfig: NextConfig = { cacheComponents: true };
export default nextConfig;
```

```tsx
import { cacheLife, cacheTag } from "next/cache";

// Cache a data function
export async function getProducts(category: string) {
  "use cache";
  cacheLife("hours");                 // built-in profiles: "seconds" | "minutes" | "hours" | "days" | "weeks" | "max" (or custom)
  cacheTag("products", `category:${category}`);
  return db.product.findMany({ where: { category } });   // arguments become part of the cache key
}

// Cache a whole component's output
async function PopularPosts() {
  "use cache";
  cacheLife("days");
  const posts = await getPopularPosts();
  return <PostList posts={posts} />;
}

// Page: static shell + dynamic holes
export default function Page() {
  return (
    <>
      <PopularPosts />                              {/* cached → part of the static shell */}
      <Suspense fallback={<GreetingSkeleton />}>
        <Greeting />                                {/* reads cookies → dynamic, streamed */}
      </Suspense>
    </>
  );
}
```

Rules:
- `"use cache"` can be applied at **file**, **component** or **function** level.
- Arguments and closed-over values become part of the **cache key**.
- You **can't** read `cookies()`/`headers()` inside a cached scope — read them outside and pass values in as arguments (or use `"use cache: private"` for per-user caching where supported).
- Uncached data access outside `<Suspense>` triggers a build error, guiding you to either cache it or stream it.

## 7.5 Revalidation APIs ⭐

| API | Where | Effect |
| --- | --- | --- |
| `revalidatePath("/blog")` | Server Actions, Route Handlers | Invalidate cached data & route for a path (`"layout"` type to include children) |
| `revalidateTag("posts", "max")` 🆕 | Server Actions, Route Handlers | Mark tagged entries stale (**stale-while-revalidate**: next visitor may see stale once, then fresh). Next 16 requires a cacheLife profile as the 2nd argument |
| `updateTag("posts")` 🆕 | Server Actions only | **Expire immediately** — the user who made the change sees fresh data in the same request (read-your-own-writes) |
| `refresh()` 🆕 | Server Actions | Refresh uncached/dynamic data on the client without touching the cache |
| `router.refresh()` | Client | Re-fetch current route's RSC payload (clears router cache for it) |
| Time-based (`revalidate`, `cacheLife`) | Data/segment config | Regenerate after the interval |

```ts
// app/api/revalidate/route.ts — CMS webhook triggers on-demand ISR
import { revalidateTag } from "next/cache";
import { NextResponse, type NextRequest } from "next/server";

export async function POST(req: NextRequest) {
  if (req.headers.get("x-webhook-secret") !== process.env.REVALIDATE_SECRET) {
    return NextResponse.json({ message: "Unauthorized" }, { status: 401 });
  }
  const { tag } = await req.json();
  revalidateTag(tag, "max");
  return NextResponse.json({ revalidated: true, now: Date.now() });
}
```

## 7.6 Opting out & debugging

- Dynamic data: `cache: "no-store"`, `connection()`, dynamic APIs, or simply don't add `"use cache"` (Cache Components).
- Inspect the build output (○ ● ƒ ◐), `NEXT_PRIVATE_DEBUG_CACHE=1`, logging config (`logging.fetches.fullUrl`).
- Self-hosting multiple instances: configure a **shared cache handler** (e.g. Redis) so ISR/data cache is consistent across instances.

## 7.7 How to explain caching in an interview (template)

> "Next.js has four caching layers: request memoization dedupes identical fetches within one render; the data cache persists fetch results across requests until revalidated; the full route cache stores the rendered HTML and RSC payload of static routes; and the client router cache keeps visited segments in the browser for instant navigation. Routes are static by default unless they use request-time APIs like cookies or search params. I revalidate time-based with `revalidate`/`cacheLife`, or on demand with `revalidateTag`/`revalidatePath` after mutations or CMS webhooks. Defaults changed in Next 15 — fetch is no longer cached by default — and Next 16's Cache Components make caching explicit with `"use cache"`, where everything is dynamic unless cached, enabling Partial Prerendering."

---

# 8. Mutations: Server Actions & Route Handlers ⭐

## 8.1 Server Actions ⭐

Async functions that run **on the server**, invoked from forms or client code. Marked with `"use server"`. Next.js creates the POST endpoint for you.

```ts
// app/todos/actions.ts
"use server";

import { z } from "zod";
import { revalidatePath } from "next/cache";
import { redirect } from "next/navigation";
import { auth } from "@/lib/auth";
import { db } from "@/lib/db";

const TodoSchema = z.object({ title: z.string().trim().min(1, "Title is required").max(200) });

export type FormState = { error?: string; success?: boolean };

export async function createTodo(prevState: FormState, formData: FormData): Promise<FormState> {
  const session = await auth();                         // 1. authenticate
  if (!session?.user) return { error: "Unauthorized" };

  const parsed = TodoSchema.safeParse({ title: formData.get("title") });   // 2. validate
  if (!parsed.success) return { error: parsed.error.issues[0]?.message };

  await db.todo.create({ data: { title: parsed.data.title, userId: session.user.id } });  // 3. mutate (authorized)
  revalidatePath("/todos");                             // 4. refresh cached UI
  return { success: true };
}

export async function deleteTodo(id: string) {
  const session = await auth();
  if (!session?.user) throw new Error("Unauthorized");
  await db.todo.delete({ where: { id, userId: session.user.id } });   // ownership check
  revalidatePath("/todos");
}
```

### Using them

```tsx
// Progressive enhancement: works even before JS loads when rendered by a Server Component
import { deleteTodo } from "./actions";

export function DeleteForm({ id }: { id: string }) {
  return (
    <form action={deleteTodo.bind(null, id)}>     {/* bind extra args */}
      <button type="submit">Delete</button>
    </form>
  );
}
```

```tsx
// Client component with pending + error state (React 19 hooks)
"use client";
import { useActionState } from "react";
import { useFormStatus } from "react-dom";
import { createTodo, type FormState } from "./actions";

function SubmitButton() {
  const { pending } = useFormStatus();
  return <button disabled={pending}>{pending ? "Adding…" : "Add"}</button>;
}

export function NewTodoForm() {
  const [state, formAction] = useActionState<FormState, FormData>(createTodo, {});
  return (
    <form action={formAction}>
      <input name="title" aria-label="Todo title" />
      <SubmitButton />
      {state.error && <p role="alert">{state.error}</p>}
    </form>
  );
}
```

Also callable from event handlers (`startTransition(() => deleteTodo(id))`), with **`useOptimistic`** for instant UI.

### How they work

- At build time, each action gets an **ID**; the client calls `POST /current-route` with that ID and serialized arguments.
- Actions run **sequentially** per client (queued) and are designed for **mutations**, not data fetching.
- The response can include updated RSC payload for revalidated routes → UI updates in one round trip.

### Server Action security ⚠️⭐

- They're **public HTTP endpoints** — anyone can call them with any arguments.
- **Always authenticate and authorize inside the action**, validate inputs (Zod), check resource ownership.
- Next.js protects against CSRF by comparing `Origin` and `Host` headers and only allowing POST.
- Closures/bound args are encrypted, but don't rely on that for authorization.
- Return only what the client needs.
- Default body size limit (1 MB) — configurable via `serverActions.bodySizeLimit`.

## 8.2 Route Handlers

API endpoints using Web `Request`/`Response`:

```ts
// app/api/products/route.ts
import { NextResponse, type NextRequest } from "next/server";

export async function GET(request: NextRequest) {
  const limit = Number(request.nextUrl.searchParams.get("limit") ?? 20);
  const products = await db.product.findMany({ take: limit });
  return NextResponse.json(products, {
    headers: { "Cache-Control": "public, s-maxage=60, stale-while-revalidate=300" },
  });
}

export async function POST(request: NextRequest) {
  const body = await request.json();
  const product = await db.product.create({ data: body });   // validate + authorize in real code
  return NextResponse.json(product, { status: 201 });
}
```

```ts
// app/api/products/[id]/route.ts
export async function GET(_req: Request, { params }: { params: Promise<{ id: string }> }) {
  const { id } = await params;
  return Response.json(await db.product.findUnique({ where: { id } }));
}
```

Supported methods: `GET`, `POST`, `PUT`, `PATCH`, `DELETE`, `HEAD`, `OPTIONS`.

## 8.3 Server Actions vs Route Handlers ⭐

| | Server Actions | Route Handlers |
| --- | --- | --- |
| Purpose | Mutations from **your own UI** | Public/external APIs |
| Invocation | Forms, `startTransition`, direct function call | Any HTTP client (`fetch`, mobile apps, webhooks) |
| HTTP method | POST only (auto-generated endpoint) | Any method, custom URL |
| Integration | Revalidate + return updated UI in one round trip; progressive enhancement | Manual |
| Use for | Create/update/delete from forms & buttons | Webhooks (Stripe, CMS), mobile/3rd-party clients, file streaming, OAuth callbacks, RSS |

---

# 9. Proxy (Middleware)

Code that runs **before** a request is completed — to redirect, rewrite, set headers/cookies or block.

🆕 **Next 16 renamed `middleware.ts` → `proxy.ts`** (function `proxy`), running on the Node.js runtime. `middleware.ts` (Edge runtime) is deprecated but still works for Edge use cases.

```ts
// proxy.ts (project root or src/)
import { NextResponse, type NextRequest } from "next/server";

export function proxy(request: NextRequest) {
  const session = request.cookies.get("session")?.value;
  const { pathname } = request.nextUrl;

  // Optimistic auth redirect
  if (!session && pathname.startsWith("/dashboard")) {
    const url = new URL("/login", request.url);
    url.searchParams.set("callbackUrl", pathname);
    return NextResponse.redirect(url);
  }

  // Add a header
  const response = NextResponse.next();
  response.headers.set("x-pathname", pathname);
  return response;
}

export const config = {
  matcher: ["/((?!api|_next/static|_next/image|favicon.ico|sitemap.xml|robots.txt).*)"],
};
```

### Use cases

- Auth redirects (optimistic checks), role-based routing
- i18n locale detection & redirects
- A/B tests & feature flags (rewrite to variants)
- Geolocation-based content
- Rewrites to legacy apps / microfrontends
- Security headers, bot protection, rate limiting (with external store)

### Limitations / best practices

- Runs on **every matched request** → keep it fast; use `matcher` to limit scope.
- Don't do slow DB queries here; do **optimistic** checks (cookie exists, JWT signature), and **authorize again** in Server Components/Actions/Route Handlers.
- ⚠️ Never rely on middleware/proxy alone for authorization (a 2025 CVE allowed bypassing middleware via a crafted header in older versions — keep Next.js patched and defend in depth).

---

# 10. Authentication & Authorization ⭐

### Recommended architecture

```text
1. Session stored in an HttpOnly, Secure, SameSite cookie (session ID or encrypted JWT)
2. Proxy/middleware: optimistic redirect for unauthenticated users (fast UX)
3. Data Access Layer (DAL): verifySession() + authorization checks for every data access
4. Server Components: call the DAL; render only what the user may see
5. Server Actions & Route Handlers: authenticate + authorize EVERY call
6. DTOs: return only safe fields to the client
```

```ts
// lib/dal.ts
import "server-only";
import { cache } from "react";
import { cookies } from "next/headers";
import { redirect } from "next/navigation";

export const verifySession = cache(async () => {
  const token = (await cookies()).get("session")?.value;
  const session = token ? await decrypt(token) : null;
  if (!session?.userId) redirect("/login");
  return { userId: session.userId, role: session.role };
});

export async function getInvoices() {
  const { userId } = await verifySession();
  return db.invoice.findMany({ where: { userId }, select: { id: true, amount: true, status: true } }); // DTO
}
```

Libraries: **Auth.js (NextAuth v5)**, **Clerk**, **Better Auth**, Supabase Auth, Firebase Auth, WorkOS, Kinde.

⚠️ **Layouts don't re-render on navigation** — don't put the only auth check in a layout; check close to the data (DAL) and in pages.

---

# 11. Optimization ⭐⭐⭐

## 11.1 Core Web Vitals → Next.js features

| Metric | Measures | Good | Next.js levers |
| --- | --- | --- | --- |
| **LCP** (Largest Contentful Paint) | Loading of main content | ≤ 2.5 s | SSG/ISR/PPR, streaming, `next/image` with `preload`/`priority` & `sizes`, `next/font`, CDN caching, fewer render-blocking resources |
| **INP** (Interaction to Next Paint) | Responsiveness | ≤ 200 ms | Server Components (less JS), code splitting, `next/dynamic`, deferring third-party scripts, `useTransition`, React Compiler |
| **CLS** (Cumulative Layout Shift) | Visual stability | ≤ 0.1 | `next/image` (reserved dimensions), `next/font` (no font swap shift), skeletons with fixed sizes |
| **TTFB** | Server response time | ≤ 0.8 s | Static/cached routes, edge/CDN, streaming, avoiding waterfalls, regions near your DB |

```tsx
// Measure in production
"use client";
import { useReportWebVitals } from "next/web-vitals";

export function WebVitals() {
  useReportWebVitals((metric) => {
    navigator.sendBeacon("/api/vitals", JSON.stringify(metric));   // or analytics
  });
  return null;
}
```

Vercel Speed Insights / Analytics, Lighthouse, PageSpeed Insights (field data from CrUX).

## 11.2 Image optimization: `next/image` ⭐

```tsx
import Image from "next/image";
import hero from "@/public/hero.jpg";   // static import → width/height/blur placeholder automatic

export function Hero() {
  return (
    <Image
      src={hero}
      alt="Person using the app on a laptop"
      priority                       // LCP image: preload, no lazy loading (Next 16: `preload` prop also available)
      placeholder="blur"
      sizes="100vw"
      style={{ width: "100%", height: "auto" }}
    />
  );
}

// Remote images need width/height (or fill) and allowed hosts
<Image src="https://cdn.example.com/p/123.jpg" alt="Red sneakers" width={600} height={600}
       sizes="(max-width: 768px) 50vw, 300px" />

// Fill a parent container
<div style={{ position: "relative", aspectRatio: "16/9" }}>
  <Image src={url} alt="" fill sizes="(max-width: 768px) 100vw, 50vw" style={{ objectFit: "cover" }} />
</div>
```

```ts
// next.config.ts
images: {
  remotePatterns: [{ protocol: "https", hostname: "cdn.example.com", pathname: "/p/**" }],
  formats: ["image/avif", "image/webp"],
}
```

**What it does:**
- Generates **responsive `srcset`** in multiple sizes; serves **modern formats** (WebP/AVIF)
- **Lazy-loads** offscreen images by default (`loading="lazy"`)
- **Prevents CLS** by requiring dimensions (or `fill` with a sized parent)
- Optional **blur placeholder**
- Caches optimized images

**Tips:** always set **`sizes`** for responsive layouts (otherwise the browser may download oversized images); mark the **LCP image** `priority`; meaningful `alt` text (a11y + SEO); for static export or external CDNs (Cloudinary, Imgix) configure a custom `loader`.

## 11.3 Font optimization: `next/font` ⭐

```tsx
// app/layout.tsx
import { Inter, Roboto_Mono } from "next/font/google";
import localFont from "next/font/local";

const inter = Inter({ subsets: ["latin"], display: "swap", variable: "--font-inter" });
const mono = Roboto_Mono({ subsets: ["latin"], variable: "--font-mono" });
const brand = localFont({ src: "./fonts/Brand.woff2", variable: "--font-brand" });

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="en" className={`${inter.variable} ${mono.variable} ${brand.variable}`}>
      <body className="font-sans">{children}</body>
    </html>
  );
}
```

- Fonts are **downloaded at build time and self-hosted** → no request to Google at runtime (privacy, performance).
- Automatic **preloading**, subsetting, and **size-adjusted fallback fonts** → near-zero CLS from font swapping.

## 11.4 Script optimization: `next/script`

```tsx
import Script from "next/script";

<Script src="https://www.googletagmanager.com/gtag/js?id=G-XXX" strategy="afterInteractive" />
<Script src="https://widget.example.com/chat.js" strategy="lazyOnload" />
<Script id="consent" strategy="beforeInteractive" src="/consent.js" />   // root layout only
```

| Strategy | Loads |
| --- | --- |
| `beforeInteractive` | Before hydration (critical scripts only: consent, bot detection) |
| `afterInteractive` (default) | After hydration (analytics, tag managers) |
| `lazyOnload` | During browser idle time (chat widgets, social embeds) |
| `worker` (experimental) | In a web worker via Partytown |

`@next/third-parties` provides optimized components for Google Analytics, Tag Manager, Maps, YouTube embeds.

## 11.5 Reducing JavaScript ⭐

- **Server Components by default** — keep data fetching & heavy rendering on the server.
- **Push `"use client"` to leaves**.
- **Lazy-load** heavy client components:

```tsx
import dynamic from "next/dynamic";

const Chart = dynamic(() => import("@/components/Chart"), {
  loading: () => <ChartSkeleton />,
  ssr: false,          // only in Client Components: render purely on the client (uses window)
});
const RichEditor = dynamic(() => import("@/components/RichEditor"));   // separate chunk, loaded when rendered
```

- Lazy-load libraries on interaction: `const { jsPDF } = await import("jspdf")`.
- Avoid barrel-file bloat; Next optimizes common packages (`optimizePackageImports`).
- Analyze bundles: `@next/bundle-analyzer` (webpack) or `next experimental-analyze` / Turbopack bundle analysis tooling.
- **React Compiler** (`reactCompiler: true`) — automatic memoization reduces unnecessary re-renders.

## 11.6 Rendering & data optimizations

- Prefer **static** (SSG/ISR/cached components) whenever content isn't per-request.
- Use **PPR / Cache Components** to keep pages static while streaming dynamic parts.
- **Stream** slow sections with Suspense; `loading.tsx` for instant feedback.
- **Parallelize** data fetching; `React.cache` to dedupe.
- **Prefetching** via `<Link>` (default in viewport); `router.prefetch()` for programmatic.
- Choose regions close to your database; edge for lightweight logic.
- `after()` to run work (logging, analytics) **after** the response is sent:

```ts
import { after } from "next/server";
export async function POST(req: Request) {
  const data = await req.json();
  after(() => logAnalytics(data));   // doesn't delay the response
  return Response.json({ ok: true });
}
```

## 11.7 Caching headers & CDN

- Static assets under `/_next/static` are content-hashed and served with immutable caching.
- Static routes and ISR are cached on the CDN (Vercel or your own with proper `Cache-Control`).
- Configure custom headers in `next.config.ts` → `headers()`.

## 11.8 Optimization checklist ⭐

- [ ] Static/ISR/cached by default; dynamic only where needed
- [ ] Suspense streaming for slow data; no waterfalls
- [ ] `next/image` with `sizes`; `priority` on LCP image; AVIF/WebP
- [ ] `next/font`; no external font CSS
- [ ] Third-party scripts via `next/script` with the right strategy
- [ ] `"use client"` only at leaves; heavy components lazy-loaded
- [ ] Bundle analyzed; large deps replaced or split
- [ ] Web Vitals monitored in production
- [ ] Caching/revalidation strategy documented
- [ ] Images & fonts have explicit dimensions → CLS ≈ 0

---

# 12. SEO ⭐⭐⭐

## 12.1 Why Next.js is good for SEO ⭐

- **Server-rendered HTML** (SSG/SSR/ISR/RSC): crawlers receive full content in the initial HTML — no reliance on executing JavaScript (Google can run JS, but with delays; many other crawlers and social preview bots can't).
- **Metadata API** for titles, descriptions, Open Graph, canonical, robots — rendered in `<head>` on the server.
- **File-based** `sitemap.xml`, `robots.txt`, OG images.
- **Performance** (Core Web Vitals) is a ranking signal — Next.js optimizations help.
- **Proper status codes** (404 via `notFound()`, 301/308 redirects) and **crawlable links** (`<Link>` renders `<a href>`).

## 12.2 Metadata API ⭐

### Static metadata

```tsx
// app/layout.tsx
import type { Metadata } from "next";

export const metadata: Metadata = {
  metadataBase: new URL("https://www.acme.com"),          // resolves relative URLs (OG images, canonical)
  title: {
    default: "Acme — Sustainable Sneakers",
    template: "%s | Acme",                                 // child pages: "Product name | Acme"
  },
  description: "Comfortable sneakers made from recycled materials.",
  keywords: ["sneakers", "sustainable shoes"],              // minor value for Google
  authors: [{ name: "Acme Team" }],
  openGraph: {
    type: "website",
    siteName: "Acme",
    locale: "en_US",
    images: ["/og-default.png"],
  },
  twitter: { card: "summary_large_image", creator: "@acme" },
  robots: { index: true, follow: true },
  alternates: { canonical: "/" },
  verification: { google: "google-site-verification-code" },
};
```

### Dynamic metadata ⭐

```tsx
// app/products/[slug]/page.tsx
import type { Metadata } from "next";
import { notFound } from "next/navigation";

type Props = { params: Promise<{ slug: string }> };

export async function generateMetadata({ params }: Props): Promise<Metadata> {
  const { slug } = await params;
  const product = await getProduct(slug);          // deduped with the page's call (fetch memo / React.cache)
  if (!product) return { title: "Product not found" };

  return {
    title: product.name,                           // → "Red Runner | Acme" via template
    description: product.summary.slice(0, 155),
    alternates: { canonical: `/products/${slug}` },
    openGraph: {
      title: product.name,
      description: product.summary,
      type: "website",
      images: [{ url: product.imageUrl, width: 1200, height: 630, alt: product.name }],
    },
  };
}

export default async function ProductPage({ params }: Props) {
  const { slug } = await params;
  const product = await getProduct(slug);
  if (!product) notFound();
  return <ProductView product={product} />;
}
```

- Metadata **merges** from root layout → nested layouts → page (deeper overrides).
- Only in **Server Components** (`metadata` / `generateMetadata` can't be used in Client Components).
- 🆕 Next 15.2+ can **stream metadata** for faster TTFB in browsers, while **bots/crawlers** (detected by user agent) receive metadata in the initial `<head>`.

## 12.3 Canonical URLs & duplicate content ⭐

```ts
alternates: {
  canonical: "/products/red-runner",                       // one preferred URL per content
  languages: { "en-US": "/en/products/red-runner", "hi-IN": "/hi/products/red-runner" }, // hreflang
}
```

Avoid duplicates from query params (`?ref=`, `?sort=`), trailing slashes, `www` vs non-`www`, HTTP vs HTTPS — choose one and redirect/canonicalize.

## 12.4 Sitemap ⭐

```ts
// app/sitemap.ts → /sitemap.xml
import type { MetadataRoute } from "next";

export default async function sitemap(): Promise<MetadataRoute.Sitemap> {
  const base = "https://www.acme.com";
  const products = await db.product.findMany({ select: { slug: true, updatedAt: true } });

  return [
    { url: base, lastModified: new Date(), changeFrequency: "daily", priority: 1 },
    { url: `${base}/about`, lastModified: new Date("2026-01-10"), priority: 0.5 },
    ...products.map((p) => ({
      url: `${base}/products/${p.slug}`,
      lastModified: p.updatedAt,
      changeFrequency: "weekly" as const,
      priority: 0.8,
    })),
  ];
}
```

Large sites (> 50,000 URLs per sitemap limit): use `generateSitemaps()` to split into multiple sitemaps.

## 12.5 robots.txt

```ts
// app/robots.ts → /robots.txt
import type { MetadataRoute } from "next";

export default function robots(): MetadataRoute.Robots {
  const isProd = process.env.VERCEL_ENV === "production";
  return {
    rules: isProd
      ? [{ userAgent: "*", allow: "/", disallow: ["/api/", "/dashboard/", "/checkout/"] }]
      : [{ userAgent: "*", disallow: "/" }],                // block staging/preview
    sitemap: "https://www.acme.com/sitemap.xml",
  };
}
```

⚠️ `robots.txt` controls **crawling**, not **indexing**. To keep a page out of search results, use `robots: { index: false }` (meta `noindex`) — and don't block it in robots.txt, or Google can't see the noindex.

## 12.6 Open Graph images (social previews) ⭐

```tsx
// app/products/[slug]/opengraph-image.tsx → auto-linked og:image for this route
import { ImageResponse } from "next/og";

export const size = { width: 1200, height: 630 };
export const contentType = "image/png";
export const alt = "Product preview";

export default async function OgImage({ params }: { params: Promise<{ slug: string }> }) {
  const { slug } = await params;
  const product = await getProduct(slug);
  return new ImageResponse(
    (
      <div style={{ display: "flex", width: "100%", height: "100%", alignItems: "center",
                    justifyContent: "center", fontSize: 64, background: "#0f172a", color: "white" }}>
        {product?.name ?? "Acme"}
      </div>
    ),
    size,
  );
}
```

Static alternatives: put `opengraph-image.png`, `twitter-image.png`, `icon.png`, `apple-icon.png`, `favicon.ico` in the route folder — Next.js generates the tags.

## 12.7 Structured data (JSON-LD) ⭐

Helps search engines understand content → **rich results** (stars, prices, breadcrumbs, FAQs).

```tsx
export default async function ProductPage({ params }: { params: Promise<{ slug: string }> }) {
  const { slug } = await params;
  const product = await getProduct(slug);
  if (!product) notFound();

  const jsonLd = {
    "@context": "https://schema.org",
    "@type": "Product",
    name: product.name,
    image: product.imageUrl,
    description: product.summary,
    sku: product.sku,
    offers: {
      "@type": "Offer",
      price: product.price,
      priceCurrency: "INR",
      availability: product.inStock ? "https://schema.org/InStock" : "https://schema.org/OutOfStock",
    },
    aggregateRating: { "@type": "AggregateRating", ratingValue: product.rating, reviewCount: product.reviewCount },
  };

  return (
    <>
      <script
        type="application/ld+json"
        dangerouslySetInnerHTML={{ __html: JSON.stringify(jsonLd).replace(/</g, "\\u003c") }} // escape to prevent XSS
      />
      <ProductView product={product} />
    </>
  );
}
```

Validate with Google's Rich Results Test. Common types: `Organization`, `WebSite` (sitelinks search box), `BreadcrumbList`, `Article`, `Product`, `FAQPage`, `LocalBusiness`, `Event`.

## 12.8 Status codes, redirects & URLs

```ts
// next.config.ts
async redirects() {
  return [
    { source: "/old-blog/:slug", destination: "/blog/:slug", permanent: true },   // 308
    { source: "/promo", destination: "/sale", permanent: false },                 // 307
  ];
},
trailingSlash: false,
```

- `notFound()` → real **404** (not a "soft 404" page with 200).
- `permanentRedirect()` / `permanent: true` → preserves SEO value when URLs change.
- Large redirect lists: handle in proxy with a lookup (or Vercel's bulk redirects).
- Use **readable, stable, lowercase, hyphenated** URLs: `/products/red-runner`, not `/p?id=123`.

## 12.9 Internationalization SEO

- Locale in the URL (`/en/...`, `/hi/...`) — crawlable separate URLs.
- `hreflang` via `alternates.languages`; `x-default` for fallback.
- `<html lang="...">` set per locale.
- Don't auto-redirect crawlers purely by IP/Accept-Language without offering links to other locales.

## 12.10 SEO pitfalls in Next.js ⚠️

| Pitfall | Fix |
| --- | --- |
| Important content rendered only client-side (`useEffect` fetch, `ssr: false`) | Fetch in Server Components |
| Missing/duplicate titles & descriptions | `title.template`, `generateMetadata` per page |
| No canonical on filtered/sorted pages | `alternates.canonical` |
| Staging/preview sites indexed | `noindex` + robots disallow on non-production; password-protect previews |
| Soft 404s (200 status with "Not found" UI) | `notFound()` |
| Huge JS & slow LCP | Optimization (§11) |
| Infinite scroll without crawlable pagination | Provide paginated URLs (`?page=2`) / links |
| Links via `onClick` + `router.push` only | Use `<Link>` (real `<a href>`) |
| Images without `alt` | Descriptive `alt` |
| Blocking `/_next/` in robots.txt | Don't — crawlers need JS/CSS to render |

## 12.11 SEO checklist ⭐

- [ ] Content server-rendered (SSG/ISR/SSR/RSC)
- [ ] Unique title (50–60 chars) & description (≤ 155–160 chars) per page
- [ ] `metadataBase`, canonical URLs, hreflang for locales
- [ ] Open Graph & Twitter cards with 1200×630 images
- [ ] `sitemap.xml` (submitted to Google Search Console) and `robots.txt`
- [ ] JSON-LD structured data for key page types
- [ ] Semantic HTML: one `<h1>`, heading hierarchy, `<nav>`, `<main>`, `<article>`
- [ ] Real 404s & 301/308 redirects; clean URLs
- [ ] Fast Core Web Vitals; mobile-friendly
- [ ] `noindex` on staging, internal search results, thin pages
- [ ] Descriptive link text and image `alt`

---

# 13. Styling

| Approach | Notes in Next.js |
| --- | --- |
| **Tailwind CSS** | Most popular; zero runtime; works with RSC |
| **CSS Modules** (`*.module.css`) | Built-in, scoped, RSC-friendly |
| Global CSS | Import in root layout (`app/globals.css`) |
| Sass | Built-in once `sass` is installed |
| **CSS-in-JS** (styled-components, Emotion) | Runtime libraries require **Client Components** + a style registry for SSR; prefer zero-runtime (vanilla-extract, Panda, StyleX) with RSC |

```tsx
import styles from "./button.module.css";
export function Button() {
  return <button className={styles.primary}>Save</button>;
}
```

Dark mode without flash: `next-themes` (injects a blocking script setting the theme class before paint).

---

# 14. Configuration & Environment Variables

## 14.1 `next.config.ts`

```ts
import type { NextConfig } from "next";

const nextConfig: NextConfig = {
  reactStrictMode: true,
  reactCompiler: true,                   // automatic memoization (requires babel-plugin-react-compiler)
  cacheComponents: true,                 // Next 16: "use cache", PPR
  images: { remotePatterns: [{ protocol: "https", hostname: "cdn.example.com" }] },
  output: "standalone",                  // minimal server bundle for Docker
  // output: "export",                   // fully static site (no server features)
  // basePath: "/docs",
  async redirects() {
    return [{ source: "/old", destination: "/new", permanent: true }];
  },
  async rewrites() {
    return [{ source: "/api/legacy/:path*", destination: "https://legacy.example.com/:path*" }];
  },
  async headers() {
    return [
      {
        source: "/(.*)",
        headers: [
          { key: "X-Frame-Options", value: "DENY" },
          { key: "X-Content-Type-Options", value: "nosniff" },
          { key: "Referrer-Policy", value: "strict-origin-when-cross-origin" },
        ],
      },
    ];
  },
};

export default nextConfig;
```

## 14.2 Environment variables ⭐

| File | Loaded |
| --- | --- |
| `.env` | All environments |
| `.env.local` | All except test; **gitignored** — local secrets |
| `.env.development`, `.env.production`, `.env.test` | Specific environment |

- Variables are **server-only by default** (`process.env.DATABASE_URL` works only on the server).
- **`NEXT_PUBLIC_`** prefix → **inlined into the client bundle at build time** → public & frozen at build time.
- ⚠️ Never put secrets in `NEXT_PUBLIC_*`.
- Runtime env for Docker "build once, deploy many": read `process.env` in **dynamic** server code (evaluated at request time), or expose config via an API/Server Component — not via `NEXT_PUBLIC_`.
- Validate env vars at startup (Zod / `@t3-oss/env-nextjs`).

---

# 15. Error Handling & Observability

```tsx
// app/dashboard/error.tsx — must be a Client Component
"use client";
import { useEffect } from "react";

export default function DashboardError({ error, reset }: { error: Error & { digest?: string }; reset: () => void }) {
  useEffect(() => {
    reportError(error);   // send to Sentry etc.
  }, [error]);

  return (
    <div role="alert">
      <h2>Something went wrong loading your dashboard.</h2>
      <button onClick={() => reset()}>Try again</button>   {/* re-render the segment */}
    </div>
  );
}
```

- In production, server error messages are **hidden** from the client (only a `digest` hash) to avoid leaking secrets.
- `global-error.tsx` handles root layout errors (must include `<html>` and `<body>`).
- `not-found.tsx` + `notFound()` for 404s.
- **Expected errors** (validation, "not allowed") → **return** them from Server Actions (`{ error }`), don't throw.
- **Unexpected errors** → throw → error boundaries.

### Instrumentation

```ts
// instrumentation.ts (root)
export async function register() {
  if (process.env.NEXT_RUNTIME === "nodejs") {
    await import("./sentry.server.config");   // OpenTelemetry / Sentry setup
  }
}

export async function onRequestError(err: unknown, request: { path: string; method: string }) {
  await reportToMonitoring(err, request);       // all server errors (RSC, actions, route handlers, proxy)
}
```

---

# 16. Internationalization (i18n)

App Router has no built-in i18n routing config (Pages Router had `i18n` in config). Pattern:

```text
app/[lang]/layout.tsx
app/[lang]/page.tsx
app/[lang]/products/[slug]/page.tsx
proxy.ts → detect locale (cookie / Accept-Language) → redirect "/" to "/en"
```

```tsx
// app/[lang]/layout.tsx
export async function generateStaticParams() {
  return [{ lang: "en" }, { lang: "hi" }];
}

export default async function LangLayout({ children, params }: { children: React.ReactNode; params: Promise<{ lang: string }> }) {
  const { lang } = await params;
  return (
    <html lang={lang}>
      <body>{children}</body>
    </html>
  );
}
```

Libraries: **next-intl** (most popular for App Router), `next-i18next` (Pages Router), Lingui, Paraglide. Format dates/numbers with `Intl`.

---

# 17. Testing

| Level | Tools | Notes |
| --- | --- | --- |
| Unit / component | **Vitest** or Jest + React Testing Library | Client Components & sync Server Components test normally |
| Async Server Components | Limited support in RTL | Test the data/logic functions separately; cover the page with E2E |
| Server Actions / Route Handlers | Call as functions; mock DB/auth | Route Handlers: construct a `Request` and assert the `Response` |
| E2E | **Playwright** (or Cypress) | Best coverage for RSC, streaming, navigation, SEO tags |
| Network mocking | MSW | For client fetching and Route Handler tests |

```ts
// Playwright: SEO smoke test
import { test, expect } from "@playwright/test";

test("product page has SEO tags", async ({ page }) => {
  await page.goto("/products/red-runner");
  await expect(page).toHaveTitle(/Red Runner \| Acme/);
  await expect(page.locator('link[rel="canonical"]')).toHaveAttribute("href", /\/products\/red-runner$/);
  await expect(page.locator('meta[property="og:image"]')).toHaveCount(1);
});
```

---

# 18. Deployment

| Option | Supports | Notes |
| --- | --- | --- |
| **Vercel** | Everything (ISR, PPR, edge, image optimization, caching) | Zero-config; built by Next.js creators |
| **Node server** (`next build && next start`) | All features | Self-host on VM/container; you manage scaling & caching |
| **Docker** (`output: "standalone"`) | All features | Small image with only needed files |
| **Static export** (`output: "export"`) | Static pages only | No SSR, ISR, Server Actions, proxy, image optimization (default loader), dynamic routes without `generateStaticParams` |
| Other platforms (Netlify, AWS Amplify, Cloudflare via OpenNext, Azure) | Most features via adapters | 🆕 Next 16 introduced a Build Adapters API (alpha) for platforms |

### Dockerfile (standalone)

```dockerfile
FROM node:22-alpine AS deps
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci

FROM node:22-alpine AS build
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .
RUN npm run build

FROM node:22-alpine AS runner
WORKDIR /app
ENV NODE_ENV=production PORT=3000 HOSTNAME=0.0.0.0
COPY --from=build --chown=node:node /app/.next/standalone ./
COPY --from=build --chown=node:node /app/.next/static ./.next/static
COPY --from=build --chown=node:node /app/public ./public
USER node
EXPOSE 3000
CMD ["node", "server.js"]
```

### Self-hosting considerations

- **Shared cache** for ISR/data cache across multiple instances (custom `cacheHandler`, e.g. Redis).
- Image optimization uses `sharp` (installed automatically in recent versions).
- Put a CDN in front; respect `Cache-Control` headers Next.js sets.
- Graceful shutdown, health checks, environment variables at runtime.

---

# 19. Pages Router Reference (Legacy)

Still common in existing codebases and interviews.

```tsx
// pages/products/[slug].tsx
import type { GetStaticPaths, GetStaticProps, InferGetStaticPropsType } from "next";
import Head from "next/head";

export const getStaticPaths: GetStaticPaths = async () => {
  const products = await getTopProducts();
  return {
    paths: products.map((p) => ({ params: { slug: p.slug } })),
    fallback: "blocking",   // false → 404 for others; true → fallback UI then render; "blocking" → SSR then cache
  };
};

export const getStaticProps: GetStaticProps<{ product: Product }> = async ({ params }) => {
  const product = await getProduct(params!.slug as string);
  if (!product) return { notFound: true };
  return { props: { product }, revalidate: 60 };   // ISR
};

export default function ProductPage({ product }: InferGetStaticPropsType<typeof getStaticProps>) {
  return (
    <>
      <Head>
        <title>{product.name} | Acme</title>
        <meta name="description" content={product.summary} />
      </Head>
      <h1>{product.name}</h1>
    </>
  );
}
```

```tsx
// SSR per request
export const getServerSideProps: GetServerSideProps = async ({ req, params, query }) => {
  const user = await getUserFromCookie(req.cookies.session);
  if (!user) return { redirect: { destination: "/login", permanent: false } };
  return { props: { user } };
};
```

| Pages Router | App Router equivalent |
| --- | --- |
| `getStaticProps` | `async` Server Component (+ caching) |
| `getStaticProps` + `revalidate` | `revalidate` / `next.revalidate` / `cacheLife` |
| `getStaticPaths` | `generateStaticParams` (+ `dynamicParams`) |
| `getServerSideProps` | Dynamic Server Component (uses cookies/headers/searchParams) |
| `pages/api/*` | `app/**/route.ts` Route Handlers |
| `_app.tsx` | Root `layout.tsx` + providers |
| `_document.tsx` | Root `layout.tsx` (`<html>`, `<body>`) |
| `next/head` | Metadata API |
| `next/router` (`useRouter`) | `next/navigation` (`useRouter`, `usePathname`, `useSearchParams`) |
| `getInitialProps` | (legacy) — avoid |

**Migration strategy:** both routers coexist; move routes incrementally from `pages/` to `app/`, starting with leaf pages; shared layouts become `layout.tsx`; replace data functions with Server Components; replace `next/router` with `next/navigation`.

---

# 20. Common Errors & Fixes ⭐

| Error | Cause | Fix |
| --- | --- | --- |
| **Hydration failed / text content did not match** | Server and client render differently (dates, random, `window`, extensions, invalid nesting) | Move browser-only logic to `useEffect`; `useId`; `dynamic(..., { ssr: false })`; fix HTML nesting; `suppressHydrationWarning` for timestamps |
| **`useState`/`useEffect` only works in Client Components** | Hook used in a Server Component | Add `"use client"` to that component (or extract the interactive part) |
| **Functions cannot be passed directly to Client Components** | Passing a non-Server-Action function prop from server to client | Define the handler in the client component, or pass a Server Action |
| **`window` / `document` / `localStorage` is not defined** | Browser API used during server render | Use inside `useEffect`/handlers; `typeof window` guard only in effects; `ssr: false` dynamic import |
| **Module not found: Can't resolve 'fs'** | Server-only module imported into client code | Keep it in server files; add `import "server-only"` |
| **Dynamic server usage / route couldn't be rendered statically** | Using `cookies()`/`headers()` in a route forced static | Remove `force-static`, or move dynamic part into Suspense/dynamic route |
| **Uncached data was accessed outside of `<Suspense>`** (Cache Components) | Dynamic data in the static shell | Wrap in `<Suspense>` or add `"use cache"` |
| **`params` should be awaited** / sync dynamic APIs error | Next 15+ async request APIs (sync removed in 16) | `const { slug } = await params`; `await cookies()`; codemod `npx @next/codemod@latest` |
| **Invalid src prop … hostname is not configured** | Remote image host not allowed | Add to `images.remotePatterns` |
| **Image with src … has either width or height modified / missing `sizes`** warnings | Incorrect `next/image` usage | Provide width/height or `fill` + `sizes` |
| **Body exceeded 1 MB limit** (Server Action) | Large uploads through actions | Increase `serverActions.bodySizeLimit` or upload directly to storage via presigned URLs |
| **Stale data after mutation** | Cache not revalidated | `revalidatePath`/`revalidateTag`/`updateTag` in the action; `router.refresh()` |
| **Changes in production not showing** | Static route cached at build | Revalidate, mark dynamic, or redeploy |
| **`NEXT_PUBLIC_` var undefined in Docker** | Inlined at build time, not runtime | Provide at build, or read server-side at request time |
| **`useSearchParams()` should be wrapped in a suspense boundary** | Client hook in statically rendered page | Wrap the component using it in `<Suspense>` |
| **Text content does not match due to locale** | Server timezone/locale differs | Format on client, pass ISO strings, or fix timezone |
| **`error.tsx` doesn't catch layout errors** | Boundary is inside the layout | Put `error.tsx` in the parent segment or `global-error.tsx` |

---

# 21. Best Practices ⭐

1. **Server Components by default**; `"use client"` only for interactive leaves.
2. **Fetch data where it's used** (in Server Components); parallelize; dedupe with `React.cache`.
3. **Static first**: cache what you can (`"use cache"`/revalidate); make dynamic only what's per-request; stream with Suspense.
4. **Explicit caching strategy** — document what's cached, for how long, and what revalidates it.
5. **Server Actions for mutations** from your UI; **Route Handlers** for external APIs and webhooks.
6. **Security in depth**: authenticate/authorize in DAL, actions and route handlers — not only in proxy/layouts; validate with Zod; DTOs; `server-only`.
7. **Never expose secrets** via `NEXT_PUBLIC_`.
8. **Use built-in optimizations**: `next/image`, `next/font`, `next/script`, `<Link>`, `next/dynamic`.
9. **Metadata on every page**; sitemap, robots, canonical, OG, JSON-LD.
10. **Error boundaries & loading states** per segment; return expected errors from actions.
11. **Type everything** (typed routes, `Metadata`, `PageProps`), strict TS.
12. **Monitor**: Web Vitals, error tracking, logs via instrumentation.
13. **Keep dependencies (especially Next.js/React) patched** — framework CVEs happen.
14. **Organize by feature**; colocate; private folders for non-route files.

---

# 22. Real-World Recipes

### 1. Search page with URL state (shareable, SEO-friendly, server-rendered)

```tsx
// app/search/page.tsx
import { Suspense } from "react";

export default async function SearchPage({ searchParams }: { searchParams: Promise<{ q?: string; page?: string }> }) {
  const { q = "", page = "1" } = await searchParams;
  return (
    <>
      <SearchBox defaultValue={q} />                 {/* client: updates ?q= with router.replace (debounced) */}
      <Suspense key={`${q}-${page}`} fallback={<ResultsSkeleton />}>
        <Results query={q} page={Number(page)} />    {/* server: fetches results; key resets fallback on change */}
      </Suspense>
    </>
  );
}
```

### 2. Protected dashboard

```tsx
// app/dashboard/page.tsx
import { verifySession } from "@/lib/dal";

export default async function Dashboard() {
  const { userId } = await verifySession();    // redirects to /login if not authenticated
  const stats = await getStats(userId);
  return <StatsView stats={stats} />;
}
```

### 3. Optimistic like button with a Server Action

```tsx
"use client";
import { useOptimistic, useTransition } from "react";
import { toggleLike } from "./actions";

export function LikeButton({ postId, liked, count }: { postId: string; liked: boolean; count: number }) {
  const [optimistic, setOptimistic] = useOptimistic({ liked, count });
  const [, startTransition] = useTransition();

  return (
    <button
      aria-pressed={optimistic.liked}
      onClick={() =>
        startTransition(async () => {
          setOptimistic((s) => ({ liked: !s.liked, count: s.count + (s.liked ? -1 : 1) }));
          await toggleLike(postId);            // action revalidates; real state replaces optimistic
        })
      }
    >
      ❤️ {optimistic.count}
    </button>
  );
}
```

### 4. On-demand ISR from a CMS webhook

See §7.5 (`revalidateTag` in a Route Handler secured by a secret).

### 5. Instagram-style modal with intercepting routes

See §3.5 (`@modal/(.)photo/[id]` + `photo/[id]`).

### 6. Client-only component (map/chart)

```tsx
"use client";
import dynamic from "next/dynamic";
const Map = dynamic(() => import("./Map"), { ssr: false, loading: () => <div style={{ height: 400 }} /> });
export default function StoreLocator() {
  return <Map />;
}
```

### 7. Streaming a dashboard with independent widgets

```tsx
export default function Dashboard() {
  return (
    <div className="grid grid-cols-3 gap-4">
      <Suspense fallback={<CardSkeleton />}><Revenue /></Suspense>
      <Suspense fallback={<CardSkeleton />}><Orders /></Suspense>
      <Suspense fallback={<CardSkeleton />}><Customers /></Suspense>
    </div>
  );
}
```

### 8. Typed route params helper

```ts
// Next generates PageProps/LayoutProps types (typed routes) in recent versions:
// export default async function Page(props: PageProps<"/blog/[slug]">) { const { slug } = await props.params; }
```

### 9. Prefetch data on the server for React Query

```tsx
import { dehydrate, HydrationBoundary, QueryClient } from "@tanstack/react-query";

export default async function PostsPage() {
  const queryClient = new QueryClient();
  await queryClient.prefetchQuery({ queryKey: ["posts"], queryFn: getPosts });
  return (
    <HydrationBoundary state={dehydrate(queryClient)}>
      <PostsClient />   {/* useQuery(["posts"]) has data instantly; refetches on client as configured */}
    </HydrationBoundary>
  );
}
```

---

# 23. Cheat Sheet ⭐

### Files

| File | Purpose |
| --- | --- |
| `page.tsx` | Route UI |
| `layout.tsx` | Persistent shared UI |
| `template.tsx` | Remounting shared UI |
| `loading.tsx` | Suspense fallback |
| `error.tsx` / `global-error.tsx` | Error boundaries |
| `not-found.tsx` | 404 UI |
| `route.ts` | API endpoint |
| `proxy.ts` (16) / `middleware.ts` (≤15) | Runs before requests |
| `sitemap.ts`, `robots.ts`, `manifest.ts` | SEO/PWA files |
| `opengraph-image.tsx`, `icon.tsx` | Generated images |
| `instrumentation.ts` | Monitoring setup, `onRequestError` |

### Folders

`[id]` dynamic · `[...slug]` catch-all · `[[...slug]]` optional catch-all · `(group)` route group · `_private` excluded · `@slot` parallel route · `(.)x` intercepting route

### Rendering & caching

| Want | Do |
| --- | --- |
| Static page | Default (no dynamic APIs) / `"use cache"` |
| Static pages for dynamic params | `generateStaticParams` |
| ISR | `revalidate = N` / `fetch(..., { next: { revalidate } })` / `cacheLife()` |
| On-demand revalidation | `revalidateTag`, `revalidatePath`, `updateTag` |
| Dynamic SSR | `cookies()`, `headers()`, `searchParams`, `connection()`, `cache: "no-store"` |
| Streaming | `<Suspense>` / `loading.tsx` |
| Static shell + dynamic holes | Cache Components (`cacheComponents: true`) + Suspense |
| Client-only | `"use client"` + `dynamic(..., { ssr: false })` |

### Imports

| Import | From |
| --- | --- |
| `Link` | `next/link` |
| `Image` | `next/image` |
| `Script` | `next/script` |
| `dynamic` | `next/dynamic` |
| `useRouter`, `usePathname`, `useSearchParams`, `useParams`, `redirect`, `notFound`, `permanentRedirect` | `next/navigation` |
| `cookies`, `headers`, `draftMode` | `next/headers` |
| `revalidatePath`, `revalidateTag`, `updateTag`, `cacheLife`, `cacheTag` | `next/cache` |
| `NextResponse`, `NextRequest`, `after`, `connection` | `next/server` |
| `ImageResponse` | `next/og` |
| `Metadata`, `MetadataRoute`, `NextConfig` (types) | `next` |
| Google/local fonts | `next/font/google`, `next/font/local` |

### CLI

| Command | Purpose |
| --- | --- |
| `npx create-next-app@latest` | New project |
| `next dev` | Dev server (Turbopack) |
| `next build` | Production build (route report) |
| `next start` | Serve production build |
| `npx @next/codemod@latest <transform>` | Automated upgrades |
| `next info` | Environment info for bug reports |

---

# 24. Interview Questions — Detailed Answers ⭐

## A. Fundamentals

**1. What is Next.js and why would you choose it over a plain React SPA?**

> "Next.js is a full-stack React framework. React provides components; Next.js adds file-based routing with nested layouts, server rendering strategies — static generation, server-side rendering, incremental regeneration, streaming and partial prerendering — React Server Components, data fetching with caching, Server Actions and API routes, and built-in optimizations for images, fonts and scripts, plus SEO tooling.
>
> I'd choose it when SEO or first-load performance matters — public marketing pages, e-commerce, content sites — or when I want a full-stack app in one codebase. The server sends rendered HTML, so users see content before JavaScript loads, and Server Components let me keep data fetching and heavy libraries on the server, shipping less JavaScript. A plain Vite SPA is still a fine choice for internal dashboards behind a login where SEO doesn't matter and static hosting is preferred."

**2. App Router vs Pages Router?**

> "The Pages Router is the original model: every page is a client component that's server-rendered and hydrated, with data fetched through `getStaticProps` or `getServerSideProps`, and API routes in `pages/api`. The App Router, introduced in Next 13, is built on React Server Components: components are server components by default, data is fetched directly in async components, layouts are nested and persist across navigation, and loading, error and not-found states are file conventions with streaming built in. Mutations use Server Actions and APIs use Route Handlers. The App Router is recommended for new projects; both can coexist, so migration can be incremental."

**3. What are the special files in the App Router?**

> "`page` defines a route's UI and makes it public; `layout` wraps child segments and persists across navigation; `template` is like a layout but remounts; `loading` creates an automatic Suspense boundary; `error` is a client error boundary for the segment; `global-error` catches root-layout errors; `not-found` renders for `notFound()`; `route` defines an API endpoint; and `default` is the fallback for parallel routes. There are also metadata files like `sitemap`, `robots`, `opengraph-image` and `icon`, plus root-level `proxy`/`middleware` and `instrumentation`."

**4. Layout vs template?**

> "Both wrap child routes. A layout is preserved during navigation between its children — its state and effects persist and it doesn't re-render — which is ideal for sidebars and navigation. A template creates a new instance on every navigation, so state resets and effects re-run — useful for enter animations or per-page logging."

## B. Server & Client Components

**5. What are React Server Components and how do they differ from Client Components?**

> "Server Components render only on the server — at build time or request time. They can be async, access databases and secrets directly, and their code is never sent to the browser, so they add zero JavaScript to the bundle. They can't use state, effects, event handlers or browser APIs.
>
> Client Components, marked with `'use client'`, are still pre-rendered to HTML on the server, but they're also shipped to the browser and hydrated, so they can use hooks, event handlers and browser APIs.
>
> In practice I keep pages and data fetching in Server Components and push `'use client'` to small interactive leaves like buttons, forms and widgets."

**6. What does `'use client'` actually do?**

> "It marks a boundary in the module graph: that file and everything it imports become part of the client bundle. It doesn't mean 'render only on the client' — client components are still server-side rendered for the initial HTML. You only need it at the entry point of a client subtree, not in every file."

**7. Can a Client Component render a Server Component?**

> "It can't import one, because anything imported into a client module becomes client code. But a Server Component can pass another Server Component to a Client Component as `children` or another prop. The server renders it, and the client component just places the result. That's how you wrap server-rendered content in client providers, tabs or modals."

**8. What can you pass from a Server Component to a Client Component?**

> "Only serializable values: primitives, plain objects and arrays, Dates, Maps, Sets, Promises and JSX elements. Not regular functions, class instances or symbols. The exception is Server Actions, which can be passed because they're serialized as references to server endpoints."

## C. Rendering ⭐⭐⭐

**9. Explain CSR, SSR, SSG and ISR. When would you use each?**

> "**CSR** renders in the browser: the server sends a minimal HTML shell plus JavaScript, which fetches data and renders. It's simple to host and good for highly interactive apps behind a login, but first paint is slow and it's weak for SEO.
>
> **SSR** renders HTML on the server for every request with fresh data. You get fast first content and SEO, but each request costs server time and TTFB depends on the data sources. I use it for personalized or request-dependent pages like search results or account pages.
>
> **SSG** renders at build time and serves HTML from a CDN. It's the fastest and cheapest, but content is fixed until the next build — ideal for marketing pages, docs and blogs.
>
> **ISR** is static generation that regenerates pages in the background after a revalidation interval or on demand. Visitors get cached pages at CDN speed while content stays reasonably fresh, which suits product catalogues and large blogs.
>
> In the App Router these are mostly decided by the code: routes are static unless they use request-time APIs, and revalidation settings turn static routes into ISR."

**10. How does Next.js decide whether a route is static or dynamic?**

> "By default it tries to render routes statically at build time. A route becomes dynamic when it uses request-time information: `cookies()`, `headers()`, `searchParams`, `connection()`, uncached fetches like `cache: 'no-store'`, or `export const dynamic = 'force-dynamic'`. The build output marks routes as static, SSG or dynamic. With Cache Components in Next 16 the model flips to explicit: everything is dynamic unless you cache it with `'use cache'`, and dynamic parts must sit inside Suspense so the rest can be prerendered."

**11. How does ISR work internally?**

> "It's stale-while-revalidate at the page level. The page is generated at build time or on first request and cached. Until the revalidation interval passes, every request gets the cached version. After the interval, the next request still receives the stale page immediately, but it triggers a regeneration in the background; once that succeeds the cache is updated for subsequent visitors. If regeneration fails, the last good page keeps being served. On-demand ISR skips the timer: a webhook or Server Action calls `revalidateTag` or `revalidatePath` to regenerate as soon as content changes."

**12. What is streaming and why is it useful?**

> "Streaming sends the HTML in chunks instead of waiting for all data. With Suspense boundaries — or a `loading.tsx` file — Next.js immediately sends the page shell with fallbacks, then streams each slow section's HTML as its data resolves, in the same response. This improves TTFB and First Contentful Paint, so slow data doesn't block fast data. React also hydrates sections selectively, prioritizing what the user interacts with. Crawlers still receive the full content, because it's part of the HTML response."

**13. What is Partial Prerendering?**

> "Partial Prerendering combines static and dynamic rendering in the same route. Next.js prerenders a static shell at build time — layout, navigation, cached content — and leaves holes defined by Suspense boundaries for dynamic, per-request content like a cart or personalized recommendations. On a request, the static shell is served instantly from the CDN and the dynamic parts are streamed into the holes in the same response. Before PPR, one dynamic API call made the whole route dynamic. In Next 16 it's delivered through Cache Components with `'use cache'`."

**14. What is hydration, and what causes hydration errors?**

> "Hydration is React in the browser taking server-rendered HTML for client components and attaching event listeners and state, by rendering the same tree and matching it to the existing DOM. If the client's first render differs from the server HTML, you get a hydration mismatch. Common causes are `Date.now()` or locale-formatted dates, `Math.random()`, branching on `typeof window`, reading `localStorage` during render, invalid HTML nesting like a `div` inside a `p`, and browser extensions modifying the DOM. Fixes: move browser-dependent values into `useEffect`, use `useId`, load client-only components with `dynamic` and `ssr: false`, fix the HTML, or use `suppressHydrationWarning` for unavoidable timestamp differences."

**15. Edge runtime vs Node.js runtime?**

> "The Node.js runtime supports the full Node API and npm ecosystem, so it's the default for pages, database access and heavy logic. The Edge runtime runs on a lightweight V8 isolate close to users, with fast cold starts but only a subset of Web APIs and size limits — good for simple personalization, redirects or geolocation. In Next 16, the new `proxy.ts` runs on Node.js, while the old middleware ran on Edge."

## D. Data Fetching & Caching ⭐⭐⭐

**16. How do you fetch data in the App Router?**

> "Directly in async Server Components — with `fetch` or by calling the database or ORM, since the code runs on the server. I avoid waterfalls by starting independent requests in parallel with `Promise.all`, or by splitting the page into components that each fetch their own data inside Suspense boundaries so they stream independently. I wrap shared data functions in `React.cache` so layouts, pages and `generateMetadata` don't repeat queries within a request. For client-side needs — polling, infinite scroll, user-triggered fetching — I use TanStack Query or SWR in Client Components, optionally prefetched on the server."

**17. Explain Next.js caching.**

> "There are four layers. Request memoization dedupes identical fetch or `cache()` calls during a single render. The data cache stores fetch results persistently across requests and deployments until they're revalidated. The full route cache stores the rendered HTML and RSC payload of static routes so they don't re-render. The router cache keeps RSC payloads of visited and prefetched segments in the browser for instant navigation.
>
> Revalidation is either time-based — `revalidate` or `cacheLife` — or on demand with `revalidateTag`, `revalidatePath` or `updateTag` after mutations or webhooks.
>
> The defaults changed: in Next 15, fetch requests and GET route handlers are no longer cached by default, and Next 16's Cache Components make caching explicit with `'use cache'`, `cacheLife` and `cacheTag`."

**18. `revalidatePath` vs `revalidateTag` vs `updateTag`?**

> "`revalidatePath` invalidates everything cached for a specific route path — simple when you know which page changed. `revalidateTag` invalidates all cached data tagged with a label, wherever it's used — better for granular, cross-page invalidation like 'products'. In Next 16 it uses stale-while-revalidate semantics with a cache profile argument. `updateTag` is for Server Actions when the user must see their own change immediately: it expires the tag at once so the same request renders fresh data."

**19. What is `React.cache` vs `"use cache"`?**

> "`React.cache` memoizes a function for the duration of one server request — deduping repeated calls within a render, never shared between users or requests. `'use cache'` is Next.js's persistent cache: the function or component result is stored across requests, keyed by its arguments, with a lifetime set by `cacheLife` and invalidation by `cacheTag`."

**20. How would you show fresh data after a form submission?**

> "Handle the submission with a Server Action that validates, authorizes and writes to the database, then calls `revalidatePath` or `revalidateTag` — or `updateTag` for read-your-own-writes in Next 16. Next.js returns the updated RSC payload in the same round trip, so the UI updates without a manual refetch. For instant feedback I add `useOptimistic`, and `useActionState` and `useFormStatus` for errors and pending state."

## E. Mutations, APIs & Proxy

**21. What are Server Actions and how do they work?**

> "Async functions marked with `'use server'` that run on the server but can be called from forms or client code like a normal function. Next.js gives each action an ID and generates a POST endpoint. When invoked — by a form's `action` prop or in a transition — the client sends the arguments, the server runs the function, and it can revalidate caches and return updated UI in the same response. Forms rendered by Server Components work even before JavaScript loads, which is progressive enhancement. They're for mutations, not data fetching."

**22. Are Server Actions secure?**

> "Next.js adds protections — POST-only, Origin/Host comparison against CSRF, unguessable action IDs, encrypted closure variables — but each action is still a public HTTP endpoint that anyone can call with arbitrary input. So every action must authenticate the user, authorize the specific operation (for example, check ownership in the database query), validate the input with a schema like Zod, and return only safe data. Hiding a button in the UI is not authorization."

**23. When would you use a Route Handler instead of a Server Action?**

> "When the caller isn't my own React UI: webhooks from Stripe or a CMS, a public or mobile API, OAuth callbacks, file downloads or streaming responses, RSS feeds, or anything needing custom HTTP methods, URLs or headers. Server Actions are for mutations triggered from my own components."

**24. What is Middleware/Proxy used for, and what are its limitations?**

> "It runs before a request reaches a route, so it can redirect, rewrite, set headers or cookies, or block requests — for optimistic auth redirects, locale detection, A/B tests, geolocation and security headers. Because it runs on every matched request, it must be fast. I limit it with a matcher and avoid database calls. It's not a security boundary on its own: I re-check authorization in Server Components, Server Actions and Route Handlers. In Next 16 it's called `proxy.ts` and runs on Node.js."

## F. Optimization ⭐⭐⭐

**25. How does Next.js optimize performance out of the box?**

> "Automatic code splitting per route; Server Components that ship no JavaScript; prefetching of linked routes in the viewport; static rendering and caching with CDN delivery; streaming with Suspense; `next/image` for responsive, lazy-loaded, modern-format images without layout shift; `next/font` for self-hosted fonts with no layout shift; `next/script` for loading strategies; Turbopack for fast builds; and support for the React Compiler for automatic memoization."

**26. How does `next/image` improve performance?**

> "It generates responsive image sizes and a `srcset`, serves modern formats like AVIF and WebP, lazy-loads offscreen images by default, and requires dimensions — or `fill` inside a sized container — so space is reserved and there's no layout shift. It can show blur placeholders and caches optimized versions. I set `sizes` accurately so the browser downloads the right size, mark the hero image with `priority` (or `preload`) so it isn't lazy-loaded and improves LCP, whitelist remote hosts with `remotePatterns`, and always write meaningful `alt` text."

**27. How does `next/font` work and why use it?**

> "It downloads Google or local fonts at build time and self-hosts them with your static assets, so there are no runtime requests to Google — better for performance and privacy. It preloads the font, subsets it, and generates a size-adjusted fallback font so swapping from fallback to web font causes almost no layout shift, which helps CLS."

**28. How would you improve a Next.js page with poor LCP?**

> "First, measure which element is the LCP and what delays it, using Lighthouse and field data. Then:
> 1. Make the route static or cached if possible, or use PPR, so HTML comes from the CDN quickly.
> 2. Stream slow sections with Suspense so they don't block the main content.
> 3. If the LCP element is an image, use `next/image` with `priority`, correct `sizes` and modern formats.
> 4. Use `next/font` to avoid font delays.
> 5. Defer third-party scripts with `next/script`.
> 6. Remove request waterfalls on the server.
> 7. Reduce client JavaScript blocking the main thread."

**29. How do you reduce JavaScript bundle size in Next.js?**

> "Keep components on the server and push `'use client'` to the leaves; lazy-load heavy client components with `next/dynamic` and libraries with dynamic `import()` on interaction; analyze the bundle and replace heavy dependencies; avoid importing large barrel files into client components (Next optimizes known packages); load third-party scripts with appropriate strategies; and render static content such as Markdown on the server."

**30. What is `next/dynamic` and when do you use `ssr: false`?**

> "`next/dynamic` is lazy loading — `React.lazy` plus Suspense with a loading component. It splits a component into its own chunk loaded when rendered. `ssr: false` skips server rendering entirely for components that depend on browser APIs, like maps, charts or editors that touch `window`. It's only allowed in Client Components."

## G. SEO ⭐⭐⭐

**31. Why is Next.js considered SEO-friendly?**

> "It delivers fully rendered HTML from the server through static generation, SSR or Server Components, so crawlers and social preview bots see the content without executing JavaScript. It has a Metadata API for titles, descriptions, canonical URLs, robots directives, Open Graph and Twitter cards, plus file conventions for sitemaps, robots.txt and generated OG images. It returns proper 404 and redirect status codes, `<Link>` renders crawlable anchor tags, and its performance optimizations improve Core Web Vitals, which are a ranking signal."

**32. How do you manage metadata in the App Router?**

> "Export a static `metadata` object from layouts and pages, or an async `generateMetadata` function when metadata depends on data such as a product name. Metadata merges from the root layout down, with deeper segments overriding. I set `metadataBase` and a title template in the root layout, then per-page titles, descriptions, canonical URLs via `alternates`, Open Graph images and robots rules. Data fetched in `generateMetadata` is deduped with the page's fetch through request memoization or `React.cache`. Metadata only works in Server Components."

**33. How do you generate a sitemap and robots.txt in Next.js?**

> "With `app/sitemap.ts`, which returns an array of URLs with `lastModified`, `changeFrequency` and `priority` — it can be async and pull dynamic URLs from the database, and very large sites use `generateSitemaps` to split it. `app/robots.ts` returns rules and the sitemap URL. I disallow crawling on staging and preview environments, and I remember that `robots.txt` controls crawling, not indexing — to deindex a page I use a `noindex` robots meta tag."

**34. What is a canonical URL and why does it matter?**

> "It tells search engines which URL is the preferred version when the same content is reachable through multiple URLs — query parameters, sorting, tracking codes, trailing slashes, or www versus non-www. Without it, ranking signals get split and duplicates may be indexed. In Next.js I set it with `alternates.canonical`, resolved against `metadataBase`, and redirect alternate hostnames permanently."

**35. How do you add structured data in Next.js?**

> "I build a JSON-LD object following schema.org — Product, Article, BreadcrumbList, FAQPage, Organization — and render it in a `<script type='application/ld+json'>` tag inside the page Server Component, escaping `<` characters to prevent script injection. It helps search engines understand the page and can earn rich results like prices, ratings and breadcrumbs. I validate it with Google's Rich Results Test."

**36. How do you create Open Graph images dynamically?**

> "With an `opengraph-image.tsx` file in a route segment that returns an `ImageResponse` from `next/og`. You write JSX with inline styles, and it's rendered to a PNG — for example the product name and price on a branded background. Next.js automatically adds the `og:image` meta tags with size and alt text. Static images can also be dropped in as `opengraph-image.png`."

**37. What SEO mistakes do you watch for in Next.js apps?**

> "Rendering important content only on the client through effects or `ssr: false`; missing or duplicate titles and descriptions; missing canonical tags on filtered pages; staging sites getting indexed; soft 404s returning status 200; slow LCP and layout shifts; infinite scroll without crawlable pagination; navigation done with click handlers instead of links; missing image alt text; and blocking `_next` assets in robots.txt."

## H. Architecture, Security & Deployment

**38. How do you handle authentication in the App Router?**

> "Store the session in an HttpOnly, Secure, SameSite cookie, using a library like Auth.js or a custom session. Do a fast optimistic check in the proxy to redirect unauthenticated users, then verify the session in a server-only data access layer, wrapped in `React.cache`, that every page, Server Action and Route Handler calls before reading or mutating data, with authorization checks such as ownership and roles. Return DTOs with only safe fields. I don't rely on layouts for auth, because they don't re-run on every navigation, or on the proxy alone."

**39. How do you handle errors in the App Router?**

> "`error.tsx` client boundaries per segment, with a `reset` function to retry, plus `global-error.tsx` for the root layout and `not-found.tsx` with `notFound()` for 404s. In production, server error messages are replaced with a digest so secrets don't leak. Expected errors such as validation failures are returned from Server Actions as state, not thrown. Unexpected errors are thrown and reported through `instrumentation.ts` with `onRequestError` to Sentry or OpenTelemetry."

**40. How do environment variables work in Next.js?**

> "They're loaded from `.env` files, with `.env.local` for secrets that aren't committed. By default they're only available on the server. Variables prefixed with `NEXT_PUBLIC_` are inlined into the client bundle at build time, so they're public and frozen at build — never put secrets there, and for one Docker image across environments, read server-side env at request time instead."

**41. How do you deploy Next.js outside Vercel?**

> "Build and run it as a Node server with `next start`, usually in Docker using `output: 'standalone'` for a small image. For multiple instances I configure a shared cache handler, such as Redis, so ISR and the data cache stay consistent, put a CDN in front, and set up health checks and graceful shutdown. Platforms like Netlify, AWS Amplify or Cloudflare via OpenNext use adapters. A fully static site can use `output: 'export'`, which drops server features like SSR, ISR, Server Actions, the proxy and default image optimization."

**42. What changed in Next.js 15 and 16?**

> "Next 15 moved to React 19, made request APIs like `params`, `searchParams`, `cookies()` and `headers()` asynchronous, stopped caching `fetch` and GET route handlers by default, stopped reusing page segments in the client router cache by default, and added `after()`, the `<Form>` component and TypeScript config files.
>
> Next 16 made Turbopack the default bundler and introduced Cache Components — `'use cache'`, `cacheLife` and `cacheTag` — which make caching explicit and enable Partial Prerendering. It renamed middleware to `proxy.ts` on the Node.js runtime, added `updateTag` and `refresh` and changed `revalidateTag` to take a cache profile, shipped React 19.2 and stable React Compiler support, removed synchronous access to request APIs, and removed `next lint` and AMP."

**43. How would you migrate from the Pages Router to the App Router?**

> "Incrementally, since both routers coexist. Create `app/` with a root layout replicating `_app` and `_document`, then move routes one at a time, starting with simpler leaf pages:
> - `getStaticProps` and `getServerSideProps` become async Server Components with appropriate caching.
> - `getStaticPaths` becomes `generateStaticParams`.
> - `next/head` becomes the Metadata API.
> - `next/router` becomes `next/navigation`.
> - API routes become Route Handlers or Server Actions.
> - Interactive pieces get `'use client'`.
>
> I'd verify SEO tags, status codes and performance with E2E tests for each migrated route."

**44. How do you structure a large Next.js project?**

> "Routes stay thin in `app/`, using route groups for different layouts (marketing, app, admin) and private folders for colocated components. Domain logic lives in feature modules and a server-only data access layer in `lib`. A shared UI design system holds the client components. Server Actions sit per feature and are validated with Zod. Typed env config, central auth helpers, and consistent caching tags per entity complete the picture. In monorepos, shared packages for UI and config, built with Turborepo."

**45. Walk me through what happens when a user requests a page in the App Router.**

> "The request first passes through the proxy, if one matches. Next.js matches the route segments. If the route — or its static shell, with PPR — is in the full route cache, it's served immediately from the CDN or server cache. Otherwise Next renders the Server Component tree on the server: layouts and the page run, data is fetched with request memoization and the data cache, and the output is the RSC payload plus HTML for client components. The response streams, sending the shell and Suspense fallbacks first and then resolved sections. In the browser, the HTML is displayed, client component JavaScript loads, and React hydrates. Later navigations through `<Link>` fetch only the RSC payload for changed segments — often already prefetched — and keep layouts mounted, using the client router cache."

---

# 25. Practice Projects

Build these to cement the concepts (each targets specific interview topics):

1. **Blog with MDX** — SSG with `generateStaticParams`, metadata per post, sitemap, RSS Route Handler, OG images, JSON-LD `Article`.
2. **E-commerce catalogue** — ISR/Cache Components for products, dynamic cart via cookies in Suspense (PPR), search with URL params + streaming, `next/image` gallery, Product JSON-LD.
3. **SaaS dashboard** — auth with Auth.js, DAL pattern, protected routes via proxy + server checks, parallel routes for widgets, Server Actions with optimistic UI, error boundaries.
4. **Photo feed** — intercepting routes modal, infinite scroll with React Query, image optimization.
5. **Multi-language marketing site** — `[lang]` segment, next-intl, hreflang alternates, locale detection in proxy.
6. **Self-hosted deploy** — Docker standalone image, Redis cache handler, CDN, Web Vitals reporting.

---

## ✅ Final Revision Checklist

- [ ] Explain Next.js vs React SPA, App vs Pages Router
- [ ] Know all special files, dynamic/catch-all routes, groups, parallel & intercepting routes
- [ ] Explain Server vs Client Components, `"use client"` boundaries, composition & serialization rules
- [ ] Explain CSR, SSR, SSG, ISR, streaming, PPR — and choose per page
- [ ] Explain how Next decides static vs dynamic; `generateStaticParams`, `dynamicParams`
- [ ] Explain the four caching layers, version defaults (15/16), `"use cache"`, revalidation APIs
- [ ] Fetch data in parallel; dedupe with `React.cache`; stream with Suspense
- [ ] Write secure Server Actions; know when to use Route Handlers
- [ ] Use proxy/middleware appropriately; implement auth in depth
- [ ] Optimize: images, fonts, scripts, dynamic imports, bundle size, Core Web Vitals
- [ ] SEO: metadata, canonical, sitemap, robots, OG images, JSON-LD, status codes, i18n
- [ ] Handle errors, env vars, deployment (Vercel, Docker standalone, static export)
- [ ] Map Pages Router APIs to App Router equivalents
- [ ] Fix hydration errors and other common errors

> **Golden habits:** server by default · client at the leaves · static first, stream the rest · explicit caching with clear revalidation · validate and authorize every Server Action · `next/image` + `next/font` + `next/script` · metadata on every page · measure Web Vitals in production.
