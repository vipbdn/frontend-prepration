# 21 — SSR, Hydration & Server Components 🔥

> **Interview + Understanding Guide** — Roadmap Section 21
>
> Goal: Compare rendering strategies (CSR, SSR, SSG, ISR, streaming, RSC), explain hydration and fix hydration errors, and understand React Server Components vs Client Components deeply.

---

# 1. Rendering Strategies ⭐⭐⭐

| Strategy | HTML generated | When | Pros | Cons |
| --- | --- | --- | --- | --- |
| **CSR** (Client-Side Rendering) | In the browser by JS | Every visit | Simple hosting (static), rich interactivity, cheap servers | Blank page until JS loads, poor SEO without care, slow on low-end devices |
| **SSR** (Server-Side Rendering) | On the server per request | Every request | Fast first paint (FCP/LCP), SEO, personalized | Server cost, TTFB depends on data, still needs hydration JS |
| **SSG** (Static Site Generation) | At build time | Once per build | Fastest (CDN), cheap, secure | Stale until rebuild; not for per-user data |
| **ISR** (Incremental Static Regeneration) | At build + regenerated in background | After revalidate interval / on demand | Static speed + fresh-ish data | Briefly stale content; framework-specific |
| **Streaming SSR** | Server, sent in chunks | Per request | Shell shown immediately, slow parts stream in | More complex |
| **RSC** (React Server Components) | Server components render on server; client components hydrate | Build or request | Less client JS, direct data access, no waterfalls | New mental model, framework needed |

```text
CSR:  request → empty HTML + JS bundle → JS runs → fetch data → render  (content late)
SSR:  request → server fetches + renders HTML → browser shows content → JS loads → hydrate (interactive)
SSG:  build → HTML files on CDN → request → instant HTML → hydrate
```

### Choosing

- Marketing pages, blogs, docs → **SSG/ISR**
- E-commerce product pages → **ISR/SSR** (SEO + fresh prices)
- Personalized dashboards behind login → **CSR** or **SSR/RSC**
- Highly interactive tools (editors, Figma-like) → **CSR**
- Modern Next.js apps → mix per route (RSC by default, client components where interactive)

---

# 2. SSR in React

```jsx
// server.js (simplified, Express)
import { renderToPipeableStream } from "react-dom/server";

app.get("*", (req, res) => {
  const { pipe } = renderToPipeableStream(<App url={req.url} />, {
    bootstrapScripts: ["/client.js"],
    onShellReady() {
      res.setHeader("content-type", "text/html");
      pipe(res); // stream HTML as Suspense boundaries resolve
    },
    onError(err) {
      console.error(err);
    },
  });
});
```

```jsx
// client.js
import { hydrateRoot } from "react-dom/client";
hydrateRoot(document.getElementById("root"), <App url={location.pathname} />);
```

| Server API | Use |
| --- | --- |
| `renderToString` | Legacy; no streaming/Suspense data support |
| `renderToPipeableStream` | Node streams, Suspense streaming |
| `renderToReadableStream` | Web streams (Edge, Deno, Bun) |
| `prerender` / `prerenderToNodeStream` | Static generation that waits for all data |

In practice you use a **framework** (Next.js, React Router framework mode/Remix, Astro islands) rather than wiring SSR yourself.

---

# 3. Hydration ⭐⭐⭐

**Hydration** = React on the client **attaches event listeners and state** to server-rendered HTML instead of recreating the DOM.

```text
Server HTML (visible, not interactive) → JS downloads → hydrateRoot → React renders the same tree in memory,
matches it to existing DOM nodes, attaches listeners → interactive
```

The client's first render **must produce the same output** as the server, otherwise → **hydration mismatch**.

### Common causes of hydration mismatches ⭐

| Cause | Example |
| --- | --- |
| Browser-only APIs during render | `window.innerWidth`, `localStorage`, `navigator` |
| Non-deterministic values | `Date.now()`, `new Date().toLocaleString()`, `Math.random()`, random IDs |
| Locale/timezone differences | Server UTC vs client IST formatting |
| Invalid HTML nesting | `<p><div/></p>`, `<a>` inside `<a>`, `<tr>` outside `<tbody>` — browser "fixes" the HTML |
| Conditional rendering on `typeof window` | `typeof window !== "undefined" ? <A/> : <B/>` |
| Browser extensions | Injecting attributes/elements (React 19 is more tolerant) |
| Different data on server vs client | Data refetched on client returns something else |

### Fixes

```jsx
// 1. Defer browser-only values until after hydration
function Clock() {
  const [now, setNow] = useState(null);         // same on server & first client render
  useEffect(() => setNow(new Date()), []);       // client-only update after hydration
  return <span>{now ? now.toLocaleTimeString() : "--:--"}</span>;
}

// 2. Reusable "is client" hook
function useIsClient() {
  return useSyncExternalStore(() => () => {}, () => true, () => false);
}

// 3. Use useId for IDs (not Math.random)
const id = useId();

// 4. Suppress for unavoidable one-off text differences (timestamps) — sparingly
<time suppressHydrationWarning>{new Date().toLocaleString()}</time>

// 5. Client-only component (Next.js)
const Map = dynamic(() => import("./Map"), { ssr: false });

// 6. Fix invalid HTML nesting
```

### Hydration costs

Even with SSR, the **entire app's JS** must download and hydrate before it's interactive (the "uncanny valley": page looks ready but clicks don't work). Mitigations: streaming + **selective hydration**, code splitting, **islands architecture** (Astro), **Server Components** (no JS for server parts), **resumability** (Qwik — no hydration).

### Selective hydration (React 18+)

With Suspense boundaries, React hydrates parts **independently** and **prioritizes what the user clicks**. A heavy comments section doesn't block hydrating the nav.

---

# 4. Streaming SSR ⭐

```jsx
function ProductPage() {
  return (
    <Layout>
      <ProductDetails />                       {/* in the initial shell */}
      <Suspense fallback={<ReviewsSkeleton />}>
        <Reviews />                            {/* slow data: streamed later */}
      </Suspense>
    </Layout>
  );
}
```

1. Server sends the shell + fallbacks immediately (fast TTFB/FCP).
2. When `Reviews` data is ready, server streams its HTML + a tiny inline script that swaps it into place.
3. Each boundary hydrates when its code arrives.

---

# 5. React Server Components (RSC) ⭐⭐⭐

### What they are

Components that **run only on the server** (at build time or per request). Their output is serialized (the **RSC payload**) and sent to the client, where React merges it into the tree. **Their code is never sent to the browser.**

```jsx
// Server Component (default in Next.js App Router)
import db from "@/lib/db";
import LikeButton from "./LikeButton"; // a Client Component

export default async function Post({ id }) {
  const post = await db.post.findUnique({ where: { id } }); // direct DB access, secrets safe
  return (
    <article>
      <h1>{post.title}</h1>
      <Markdown content={post.body} />   {/* heavy lib stays on the server: 0 KB to client */}
      <LikeButton postId={post.id} initialLikes={post.likes} />
    </article>
  );
}
```

```jsx
// Client Component
"use client";
import { useState } from "react";

export default function LikeButton({ postId, initialLikes }) {
  const [likes, setLikes] = useState(initialLikes);
  return <button onClick={() => setLikes((l) => l + 1)}>❤️ {likes}</button>;
}
```

### Server vs Client Components ⭐

| | Server Component | Client Component (`"use client"`) |
| --- | --- | --- |
| Runs | Server only | Server (SSR pre-render) **and** client |
| JS shipped to browser | ❌ None | ✅ Yes |
| `async` / `await` in component | ✅ | ❌ (use `use()` / libraries) |
| Direct DB / filesystem / secrets | ✅ | ❌ |
| State, effects (`useState`, `useEffect`) | ❌ | ✅ |
| Event handlers (`onClick`) | ❌ | ✅ |
| Browser APIs | ❌ | ✅ |
| Context (create/consume) | ❌ (can render providers that are client components) | ✅ |

> ⚠️ "Client Component" doesn't mean "client-only rendered" — they're still SSR'd to HTML, then hydrated. `"use client"` marks the **boundary** where the client module graph starts.

### Composition rules ⭐

- Server Components can import and render Client Components.
- Client Components **cannot import** Server Components — but can receive them as **`children`/props**:

```jsx
// ✅ Server Component passes a Server Component into a Client Component
<ClientTabs>
  <ServerRenderedContent />
</ClientTabs>
```

- Props passed from Server → Client must be **serializable** (no functions, except Server Actions; no class instances). Dates, Maps, Sets, Promises are supported in the RSC protocol.
- `"use client"` at the top of a file makes **all its imports** part of the client bundle → push it to the **leaves** (small interactive components) to keep JS small.

### Benefits

- **Smaller bundles** (server-only code & dependencies never ship)
- **Direct data access** — no API layer needed for reads; no client fetch waterfalls
- **Security** — tokens/keys stay on the server
- **Streaming** with Suspense
- Automatic code splitting of client components

### RSC vs SSR (common confusion) ⭐

| SSR | RSC |
| --- | --- |
| Renders the **whole app** to HTML for the first load; then all components hydrate on the client | Some components **never** run on the client |
| All component code ships to the client | Server component code doesn't ship |
| HTML output | RSC payload (serialized tree) — can be fetched on navigation without full page reload |
| | Usually used **together with** SSR (RSC payload → SSR'd HTML for first load) |

### `server-only` / `client-only`

```js
import "server-only"; // build error if this module ends up in a client bundle (protect secrets)
```

---

# 6. Server Actions

Functions marked `"use server"` that run on the server, callable from forms or client code.

```jsx
"use server";
export async function addToCart(productId) {
  const session = await getSession();
  if (!session) throw new Error("Unauthorized");       // ⚠️ always authenticate & authorize
  await db.cart.add(session.userId, productId);
  revalidatePath("/cart");
}
```

```jsx
"use client";
import { addToCart } from "./actions";
export function AddButton({ id }) {
  const [isPending, startTransition] = useTransition();
  return (
    <button disabled={isPending} onClick={() => startTransition(() => addToCart(id))}>
      Add to cart
    </button>
  );
}
```

Security: Server Actions are **public HTTP endpoints** — validate input (Zod), check auth, and never trust client-provided IDs for authorization.

---

# 7. Islands & Other Approaches (context)

| Approach | Idea | Examples |
| --- | --- | --- |
| Islands | Static HTML with isolated interactive "islands" hydrated independently | Astro, Fresh |
| Resumability | Serialize app state so no hydration is needed | Qwik |
| Partial Prerendering (PPR) | Static shell from CDN + dynamic holes streamed | Next.js |

---

# 8. Interview Answers

**CSR vs SSR vs SSG?**

> "CSR renders everything in the browser — simple hosting but a blank page until JS loads and weaker SEO. SSR renders HTML per request on the server — fast first content and SEO, at server cost, and it still has to hydrate. SSG renders at build time and serves from a CDN — fastest and cheapest, but only for content that's the same for everyone and can be stale until rebuild; ISR regenerates static pages in the background. I'd mix them per route."

**What is hydration and what causes mismatches?**

> "Hydration is React attaching event handlers and state to server-rendered HTML by rendering the same tree on the client and matching DOM nodes. The first client render must match the server. Mismatches come from browser-only APIs, dates, random values, locale differences or invalid HTML nesting. I fix them by moving browser-dependent values into effects, using `useId`, rendering client-only components, or `suppressHydrationWarning` for unavoidable timestamps."

**Server Components vs Client Components?**

> "Server Components run only on the server, can be async and access the database directly, and send zero JavaScript to the browser. Client Components, marked with `'use client'`, are pre-rendered on the server and hydrated on the client, and can use state, effects and event handlers. Server Components can render Client Components and pass server-rendered content into them as children, but props crossing the boundary must be serializable. I keep `'use client'` at the leaves to minimize bundle size."

---

# 9. Key Takeaways

- CSR / SSR / SSG / ISR / streaming / RSC — choose per route based on SEO, freshness, personalization, interactivity.
- Hydration attaches React to server HTML; first client render must match.
- Fix mismatches: effects for browser values, `useId`, client-only rendering, valid HTML.
- Streaming SSR + selective hydration via Suspense.
- RSC: server-only components, zero client JS, direct data access; `"use client"` marks boundaries; serializable props.
- RSC ≠ SSR; they're complementary.
- Server Actions are public endpoints — validate and authorize.

## Readiness Checklist

- [ ] Compare all rendering strategies in a table from memory
- [ ] Explain hydration and fix 4 mismatch causes
- [ ] Explain streaming SSR and selective hydration
- [ ] Explain RSC vs Client Components and composition rules
- [ ] Explain RSC vs SSR
- [ ] Write a secure Server Action
