# 22 — Next.js (App Router)

> **Interview + Understanding Guide** — Roadmap Section 22
>
> Goal: Know the Next.js App Router well enough for 4-YOE interviews: file-based routing, layouts, Server/Client Components, data fetching and caching, rendering modes, Server Actions, route handlers, middleware, metadata, images/fonts, and deployment concerns.
>
> Note: Next.js changes quickly (v15 in 2024, v16 in 2025). Concepts below are stable; check the docs for exact API names in your version.

---

# 1. Why Next.js?

A full-stack React framework providing:
- File-based routing with nested layouts
- Server Components by default, Client Components opt-in
- SSR, SSG, ISR, streaming, Partial Prerendering
- Data fetching + caching + revalidation
- Server Actions and API route handlers
- Image, font, script optimization; metadata/SEO
- Middleware, i18n, edge runtime
- Zero-config bundling (Turbopack / webpack), TypeScript, ESLint

**Pages Router** (`pages/`, `getServerSideProps`, `getStaticProps`) is the older model — still supported. **App Router** (`app/`) is the modern default.

---

# 2. File-Based Routing ⭐

```text
app/
├── layout.tsx              → root layout (required: <html><body>)
├── page.tsx                → "/"
├── loading.tsx             → Suspense fallback for this segment
├── error.tsx               → error boundary for this segment ("use client")
├── not-found.tsx           → 404 UI
├── global-error.tsx        → errors in root layout
├── about/page.tsx          → "/about"
├── blog/
│   ├── page.tsx            → "/blog"
│   └── [slug]/page.tsx     → "/blog/:slug"           (dynamic segment)
├── shop/[...slug]/page.tsx → "/shop/a/b/c"           (catch-all)
├── docs/[[...slug]]/page.tsx → "/docs" and "/docs/a/b" (optional catch-all)
├── (marketing)/pricing/page.tsx → "/pricing"         (route group: no URL segment)
├── dashboard/
│   ├── layout.tsx          → persistent dashboard shell
│   ├── @analytics/page.tsx → parallel route slot
│   └── page.tsx
└── api/users/route.ts      → API endpoint (Route Handler)
```

Special files: `page`, `layout`, `template`, `loading`, `error`, `not-found`, `route`, `default`, `middleware` (root; `proxy` in v16).

### Layouts ⭐

```tsx
// app/dashboard/layout.tsx
export default function DashboardLayout({ children }: { children: React.ReactNode }) {
  return (
    <div className="flex">
      <Sidebar />                {/* doesn't re-render or lose state when navigating between dashboard pages */}
      <main>{children}</main>
    </div>
  );
}
```

- Layouts **persist** across navigations (state preserved); `template.tsx` remounts on every navigation.
- Nested layouts compose: root → dashboard → page.

### Dynamic params

```tsx
// app/blog/[slug]/page.tsx (Next 15+: params is a Promise)
export default async function BlogPost({ params }: { params: Promise<{ slug: string }> }) {
  const { slug } = await params;
  const post = await getPost(slug);
  if (!post) notFound();
  return <article>{post.title}</article>;
}

// Pre-render known paths at build time (SSG)
export async function generateStaticParams() {
  const posts = await getAllPosts();
  return posts.map((p) => ({ slug: p.slug }));
}
```

### Navigation

```tsx
import Link from "next/link";
import { useRouter, usePathname, useSearchParams } from "next/navigation"; // client components

<Link href="/blog/hello" prefetch>Read</Link>   // prefetches in viewport (production)

const router = useRouter();
router.push("/dashboard");
router.refresh(); // re-fetch Server Components for current route
```

Server-side: `redirect("/login")`, `notFound()`, `permanentRedirect()`.

### Advanced routing

- **Parallel routes** (`@slot`) — render multiple pages in one layout (dashboards, split views).
- **Intercepting routes** (`(.)photo/[id]`) — open a route in a modal while keeping the URL shareable (Instagram-style photo modal).

---

# 3. Server & Client Components in Next.js ⭐

- Everything in `app/` is a **Server Component** by default.
- Add `"use client"` for interactivity (state, effects, handlers, browser APIs).
- Push client boundaries to the **leaves**.
- Context providers must be Client Components — wrap them in a `providers.tsx` and render in the root layout:

```tsx
// app/providers.tsx
"use client";
export function Providers({ children }: { children: React.ReactNode }) {
  const [queryClient] = useState(() => new QueryClient());
  return (
    <QueryClientProvider client={queryClient}>
      <ThemeProvider>{children}</ThemeProvider>
    </QueryClientProvider>
  );
}

// app/layout.tsx (Server Component)
export default function RootLayout({ children }) {
  return (
    <html lang="en">
      <body><Providers>{children}</Providers></body>
    </html>
  );
}
```

Full concepts: `21-ssr-hydration-server-components.md`.

---

# 4. Data Fetching ⭐

### In Server Components (preferred)

```tsx
export default async function ProductsPage() {
  const products = await db.product.findMany();                 // ORM directly
  const res = await fetch("https://api.example.com/rates", {
    next: { revalidate: 3600, tags: ["rates"] },                // cache + revalidate hourly
  });
  const rates = await res.json();
  return <ProductGrid products={products} rates={rates} />;
}
```

### Avoid waterfalls — fetch in parallel

```tsx
const [user, orders] = await Promise.all([getUser(id), getOrders(id)]);
```

### Stream slow parts

```tsx
export default function Page() {
  return (
    <>
      <Header />
      <Suspense fallback={<OrdersSkeleton />}>
        <Orders />   {/* async Server Component; streams when ready */}
      </Suspense>
    </>
  );
}
```

### Dedupe with `cache`

```ts
import { cache } from "react";
export const getUser = cache(async (id: string) => db.user.findUnique({ where: { id } }));
// Called in layout AND page during one request → one DB query
```

### Client-side fetching

For user-specific, frequently changing, or interactive data: TanStack Query/SWR in Client Components (optionally hydrated from server prefetch).

---

# 5. Caching & Revalidation ⭐⭐⭐ (frequently asked, frequently changing)

Conceptual layers:

| Layer | What | Where |
| --- | --- | --- |
| **Request memoization** | Same `fetch`/`cache()` call deduped within one render | Server, per request |
| **Data cache** | `fetch` results persisted across requests/deploys | Server |
| **Full route cache** | Rendered HTML + RSC payload of static routes | Server (build/ISR) |
| **Router cache** | RSC payloads of visited/prefetched routes | Client memory |

### Revalidation

```ts
// Time-based (ISR)
fetch(url, { next: { revalidate: 60 } });
export const revalidate = 60; // segment config

// On-demand (after a mutation)
import { revalidatePath, revalidateTag } from "next/cache";
revalidatePath("/products");
revalidateTag("products");

// Opt out (always dynamic)
fetch(url, { cache: "no-store" });
export const dynamic = "force-dynamic";
```

### Static vs Dynamic rendering

A route is **static** (rendered at build, served from cache) unless it uses **dynamic APIs**: `cookies()`, `headers()`, `searchParams`, uncached fetches → then it's rendered per request.

### Version notes

- **Next 15** changed defaults: `fetch` and GET route handlers are **not cached by default** (opt in), and the client router cache doesn't reuse page segments by default.
- **Next 16** introduced **Cache Components** with the `"use cache"` directive and `cacheLife`/`cacheTag` for explicit caching, building on Partial Prerendering.

Interview tip: explain the **concepts** (memoization, data cache, full-route cache, router cache, time-based vs on-demand revalidation, static vs dynamic) and mention that defaults changed across versions.

---

# 6. Server Actions & Mutations ⭐

```tsx
// app/todos/actions.ts
"use server";
import { z } from "zod";
import { revalidatePath } from "next/cache";

const schema = z.object({ title: z.string().min(1).max(200) });

export async function createTodo(prevState: { error?: string }, formData: FormData) {
  const session = await auth();
  if (!session) return { error: "Unauthorized" };

  const parsed = schema.safeParse({ title: formData.get("title") });
  if (!parsed.success) return { error: parsed.error.issues[0].message };

  await db.todo.create({ data: { title: parsed.data.title, userId: session.user.id } });
  revalidatePath("/todos");
  return {};
}
```

```tsx
// app/todos/new-todo-form.tsx
"use client";
import { useActionState } from "react";
import { createTodo } from "./actions";

export function NewTodoForm() {
  const [state, formAction, pending] = useActionState(createTodo, {});
  return (
    <form action={formAction}>
      <input name="title" />
      <button disabled={pending}>Add</button>
      {state.error && <p role="alert">{state.error}</p>}
    </form>
  );
}
```

Works without JS (progressive enhancement) when the form is rendered by a Server Component.

---

# 7. Route Handlers (API)

```ts
// app/api/products/route.ts
import { NextResponse } from "next/server";

export async function GET(request: Request) {
  const { searchParams } = new URL(request.url);
  const products = await db.product.findMany({ take: Number(searchParams.get("limit") ?? 20) });
  return NextResponse.json(products);
}

export async function POST(request: Request) {
  const body = await request.json();
  const product = await db.product.create({ data: body });
  return NextResponse.json(product, { status: 201 });
}
```

Use for: webhooks, public APIs, third-party integrations, mobile app backends. For your own UI mutations, Server Actions are simpler.

---

# 8. Middleware (Proxy in v16)

Runs **before** a request is completed — at the edge — for redirects, rewrites, auth checks, headers, A/B testing, i18n.

```ts
// middleware.ts (root)
import { NextResponse, type NextRequest } from "next/server";

export function middleware(request: NextRequest) {
  const token = request.cookies.get("session")?.value;
  if (!token && request.nextUrl.pathname.startsWith("/dashboard")) {
    return NextResponse.redirect(new URL("/login", request.url));
  }
  return NextResponse.next();
}

export const config = { matcher: ["/dashboard/:path*"] };
```

Keep middleware light (optimistic checks); do real authorization in Server Components/Actions/route handlers too.

---

# 9. Metadata & SEO

```tsx
// Static
export const metadata = {
  title: "Products | Shop",
  description: "Browse our products",
  openGraph: { images: ["/og.png"] },
};

// Dynamic
export async function generateMetadata({ params }) {
  const { slug } = await params;
  const post = await getPost(slug);
  return { title: post.title, description: post.excerpt };
}
```

Also: `sitemap.ts`, `robots.ts`, `opengraph-image.tsx` (generated OG images).

---

# 10. Optimizations

```tsx
import Image from "next/image";
<Image src="/hero.jpg" alt="Hero" width={1200} height={600} priority sizes="100vw" />
// → responsive srcset, lazy by default, modern formats, prevents CLS

import { Inter } from "next/font/google";
const inter = Inter({ subsets: ["latin"], display: "swap" }); // self-hosted, no layout shift

import Script from "next/script";
<Script src="https://analytics.example.com" strategy="afterInteractive" />

import dynamic from "next/dynamic";
const Chart = dynamic(() => import("./Chart"), { ssr: false, loading: () => <Skeleton /> });
```

---

# 11. Environment Variables

- `.env.local` etc.; **server-only by default**.
- Only variables prefixed **`NEXT_PUBLIC_`** are inlined into the client bundle → never put secrets there.

---

# 12. Authentication (common pattern)

- Libraries: **Auth.js (NextAuth)**, Clerk, Lucia-style custom sessions, Supabase Auth.
- Session in an **HttpOnly cookie**; read with `cookies()` in Server Components/Actions.
- Protect at several layers: middleware (redirect), layouts/pages (check session), **data access layer** (authorize every query/action).

---

# 13. Pages Router vs App Router

| Pages Router | App Router |
| --- | --- |
| `pages/` | `app/` |
| All components are client components (SSR'd) | Server Components by default |
| `getServerSideProps`, `getStaticProps`, `getStaticPaths` | `async` components + `fetch` caching, `generateStaticParams` |
| `_app.js`, `_document.js` | `layout.tsx` |
| API routes `pages/api` | Route handlers `app/**/route.ts` |
| `next/router` | `next/navigation` |
| No nested layouts natively | Nested layouts, streaming, loading/error files |

---

# 14. Interview Answers

**How does data fetching work in the App Router?**

> "Server Components can be async, so I fetch directly in them — from the database or with `fetch` — in parallel with `Promise.all` to avoid waterfalls, and wrap slow parts in Suspense to stream them. Next dedupes identical requests during a render and can cache results with time-based or tag-based revalidation. After mutations — usually Server Actions — I call `revalidatePath` or `revalidateTag`. For highly interactive client data I still use TanStack Query in Client Components."

**Static vs dynamic rendering in Next.js?**

> "Routes are statically rendered at build time and served from cache unless they use request-time information like cookies, headers or search params, or uncached data — then they render per request. ISR lets static pages revalidate on a timer or on demand. Partial Prerendering combines a static shell with dynamic streamed holes."

**How do you protect routes in Next.js?**

> "Middleware for fast redirects of unauthenticated users, plus checks in layouts or pages, and — most importantly — authorization in the data access layer, Server Actions and route handlers, because those are the real endpoints."

---

# 15. Key Takeaways

- App Router: file routing, nested persistent layouts, `loading`/`error`/`not-found` files.
- Server Components by default; `"use client"` at the leaves; providers in a client wrapper.
- Fetch in async Server Components; parallelize; stream with Suspense; `cache()` to dedupe.
- Caching layers + time/on-demand revalidation; static vs dynamic routes; defaults changed in v15/v16.
- Server Actions for mutations (validate + authorize); route handlers for APIs/webhooks.
- Middleware for edge redirects/rewrites; `NEXT_PUBLIC_` for client env vars.
- `next/image`, `next/font`, `next/script`, `dynamic()` for performance.

## Readiness Checklist

- [ ] Sketch an `app/` folder with dynamic, catch-all, group, parallel routes
- [ ] Fetch data in a Server Component with streaming
- [ ] Explain the 4 caching layers and revalidation
- [ ] Build a form with a Server Action + useActionState
- [ ] Protect a dashboard with middleware + server checks
- [ ] Compare Pages vs App Router
