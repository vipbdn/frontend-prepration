# 33 — Frontend System Design 🔥🔥🔥

> **Interview + Understanding Guide** — Roadmap Section 33
>
> Goal: Answer open-ended "Design X" frontend questions in 45–60 minutes with a clear framework (RADIO), covering requirements, architecture, data model, API design, performance, accessibility, offline, security and trade-offs. Includes 6 worked designs.

---

# 1. What Interviewers Evaluate

- **Structured thinking** — not jumping into code
- **Requirements gathering** — functional + non-functional
- **Component architecture** & state design
- **Data flow & API design** (client ↔ server contract)
- **Performance** at scale (rendering, network, bundle)
- **UX details**: loading, errors, empty, optimistic, offline
- **Accessibility & i18n**
- **Trade-off reasoning** — "it depends", then decide and justify

---

# 2. The RADIO Framework ⭐

| Step | Time | Focus |
| --- | --- | --- |
| **R**equirements | ~5–10 min | Functional scope, non-functional (performance, devices, offline, a11y, i18n, scale), out of scope |
| **A**rchitecture | ~10 min | High-level components, client/server responsibilities, rendering strategy, state management |
| **D**ata model | ~5–10 min | Client-side entities, normalized store, what's server vs client state |
| **I**nterface (API) | ~5–10 min | Endpoints/GraphQL/WebSocket, request/response shapes, pagination, component props APIs |
| **O**ptimizations & deep dives | remaining | Performance, network, rendering, a11y, security, offline, observability, edge cases |

> Narrate the plan at the start: "I'll clarify requirements, sketch the architecture, define the data model and APIs, then dive into performance and edge cases."

---

# 3. Requirements Checklist

**Functional:** core user actions, primary screens, real-time needs, search/filter, create/edit flows, notifications.

**Non-functional:**

| Area | Questions |
| --- | --- |
| Users & scale | DAU? Data size (items per list)? Global audience? |
| Devices | Mobile web? Low-end devices? Slow networks? |
| Performance | Target LCP/INP? Initial load budget? |
| SEO | Public pages need indexing? |
| Offline | Must work offline / flaky networks? |
| Real-time | Updates latency? Collaboration? |
| A11y & i18n | WCAG AA? RTL languages? Locales? |
| Security & privacy | Auth, PII, permissions |
| Browser support | Evergreen only? |

---

# 4. Architecture Building Blocks

### Rendering strategy

- Public, SEO-critical (feeds, product pages): **SSR/SSG/ISR/RSC**
- Behind login, highly interactive: **CSR** (or RSC + client islands)
- Hybrid per route

### Client architecture

```text
┌─────────────────────────────── UI Components ───────────────────────────────┐
│  Pages/Routes → Feature containers → Presentational components (design system)│
└───────────────┬─────────────────────────────────────────────────┬───────────┘
                │                                                 │
     ┌──────────▼──────────┐                         ┌────────────▼────────────┐
     │ Client state store   │                         │ Server-state cache        │
     │ (Zustand/Redux/ctx)  │                         │ (TanStack Query / RTK Q)  │
     └──────────┬──────────┘                         └────────────┬────────────┘
                │                                                 │
                └────────────────┬────────────────────────────────┘
                        ┌────────▼────────┐
                        │  API / network   │  REST · GraphQL · WebSocket · SSE
                        │  layer (client)  │  auth, retries, errors, dedupe
                        └────────┬────────┘
                                 ▼
                      BFF / API gateway / CDN
```

### Communication protocols

| Need | Choice |
| --- | --- |
| CRUD | REST or GraphQL |
| Server → client push (notifications, feed updates) | **SSE** (simple, one-way, auto-reconnect) or WebSocket |
| Bidirectional real-time (chat, collaboration, games) | **WebSocket** |
| Infrequent updates | Polling / long polling |
| Collaborative editing | WebSocket + CRDTs (Yjs) / OT |

---

# 5. Common Deep-Dive Topics

### Network

- Pagination: **cursor-based** for feeds (stable with inserts) vs offset for tables with page numbers
- Caching: HTTP cache, CDN, stale-while-revalidate, TanStack Query
- Request dedupe, retries with backoff, timeouts, cancellation (AbortController)
- Debounce search; batching; prefetch next page / on hover
- Compression, HTTP/2/3, image CDN with resizing (AVIF/WebP)

### Rendering

- Virtualization for long lists
- Code splitting per route / heavy components
- Skeletons, optimistic updates, transitions
- Memoization; state colocation; avoid layout thrashing
- Web Workers for heavy computation

### Data

- Normalized client store (`byId` + ids) to update an entity once and reflect everywhere
- Optimistic updates + rollback; conflict resolution
- Local persistence (IndexedDB) for offline/drafts

### Offline / PWA

- Service worker caching strategies: cache-first (static assets), network-first (API), stale-while-revalidate
- Queue mutations offline (background sync); retry on reconnect
- Show offline status; conflict resolution

### Accessibility & i18n

- Semantic HTML, keyboard, focus management, live regions for new content
- i18n: ICU messages, `Intl` for dates/numbers/plurals, RTL layout (`dir="rtl"`, logical CSS properties)

### Security

- XSS (sanitize user content), CSRF, auth tokens in HttpOnly cookies, CSP, rate limiting feedback

### Observability

- Error monitoring (Sentry), Web Vitals RUM, analytics events, feature flags for rollout

---

# 6. Worked Design 1: News Feed (Facebook/Twitter) ⭐

**Requirements:** infinite feed of posts (text, images), like/comment, create post, new-post notifications; mobile web; fast first load; SEO not needed (logged-in).

**Architecture:**
- Feed page shell SSR'd/streamed for fast LCP or CSR with skeleton; composer and posts as components.
- Server state: `useInfiniteQuery(["feed"])`; post entities normalized so likes update everywhere.

**Data model:**

```ts
type Post = { id: string; author: UserRef; text: string; media: Media[]; createdAt: string;
              likeCount: number; likedByMe: boolean; commentCount: number };
type FeedPage = { items: Post[]; nextCursor: string | null };
```

**API:**

```text
GET  /feed?cursor=abc&limit=20         → FeedPage    (cursor pagination — no duplicates when new posts arrive)
POST /posts                             → Post
POST /posts/:id/like  | DELETE …/like   → { likeCount }
SSE  /feed/stream                        → { type: "new_posts", count }
```

**Optimizations:**
- Infinite scroll with IntersectionObserver sentinel + prefetch next page
- **Virtualize** (dynamic heights → measure with ResizeObserver; libraries like react-virtuoso)
- Images: responsive `srcset`, lazy load, placeholders with fixed aspect ratio (no CLS), CDN
- Optimistic like with rollback; debounce double-taps
- "New posts" pill instead of auto-inserting (don't shift content under the user)
- Composer: draft persisted in localStorage; upload images with progress, cancel
- Keep scroll position when navigating back (cache + scroll restoration)
- A11y: `<article>` per post, `aria-setsize/posinset` for feed (`role="feed"`), keyboard shortcuts optional

---

# 7. Worked Design 2: Autocomplete / Search ⭐

**Requirements:** suggestions as the user types, keyboard navigation, highlight, recent searches, works for millions of items server-side.

**Key decisions:**
- **Debounce** 200–300 ms; min chars 2; **AbortController** cancels stale requests (or ignore out-of-order responses via request id)
- **Client cache** (LRU `Map`) keyed by query; optionally prefix filtering of cached broader results
- Server: dedicated search endpoint (Elasticsearch/Algolia), limit 10, return highlight ranges
- Component API: `<Autocomplete fetchSuggestions renderItem onSelect minChars debounceMs />` — headless hook + UI
- Accessibility: combobox pattern (`role="combobox"`, `listbox`, `aria-activedescendant`, live region for result counts)
- Edge cases: empty/no results, errors, slow network indicator, IME composition, mobile keyboard, pasted text
- Performance: virtualize if many options; memoize rows

Implementation reference: `32-machine-coding.md` §2.

---

# 8. Worked Design 3: Chat Application (WhatsApp/Slack web) ⭐

**Requirements:** 1:1 and group chats, real-time messages, typing indicators, read receipts, history, offline send, attachments.

**Architecture:**
- **WebSocket** connection managed by a singleton service (reconnect with exponential backoff, heartbeat/ping, resubscribe on reconnect)
- Normalized store: `conversations { byId }`, `messages { [conversationId]: { byId, order } }`, `users`
- TanStack Query for history pages (reverse infinite scroll); WebSocket events update the cache

**Message lifecycle (optimistic):**

```text
User sends → message added with clientId, status "sending" → WS send
  ack { clientId, serverId, createdAt } → status "sent" → "delivered" → "read"
  failure/timeout → status "failed" → retry button
offline → queue in IndexedDB → flush on reconnect (idempotency via clientId)
```

**Details:**
- Reverse infinite scroll (load older on scroll top) preserving scroll position; virtualization (react-virtuoso supports reverse)
- Unread counts, "jump to latest", date separators
- Typing indicator: throttle "typing" events (every 2–3 s), expire after timeout
- Attachments: direct upload to object storage via pre-signed URLs, progress, thumbnails
- Notifications: Notification API + Page Visibility (only when hidden), title badge
- Security: sanitize/escape message text, linkify safely, E2E encryption discussion if asked
- A11y: `role="log"` with `aria-live="polite"` for new messages, focus management in composer

---

# 9. Worked Design 4: E-commerce Product Listing & Detail

**Requirements:** category listing with filters/sort/pagination, product detail, add to cart, SEO, fast on mobile.

- **Rendering:** listing & PDP via SSR/ISR/RSC (SEO + LCP); cart & filters interactive client components
- **URL as state:** filters, sort, page in search params (shareable, back button, crawlable)
- Faceted filters from API (`facets: { brand: [{ value, count }] }`); debounce price slider
- Images: CDN with responsive sizes, `priority` for hero image (LCP), lazy elsewhere
- Cart: client store persisted (localStorage) + server sync when logged in; optimistic add; mini-cart drawer
- Stock/price freshness: ISR with short revalidate + client revalidation on PDP
- Performance budget: JS < ~150–200 KB gz on PDP; third-party scripts deferred
- A11y: filters as forms with labels, results count announced via live region
- Analytics: impressions via IntersectionObserver, add-to-cart events

---

# 10. Worked Design 5: Collaborative Document Editor (Google Docs-lite)

- Editor: ProseMirror/Tiptap/Lexical/Slate (contentEditable abstraction) — don't build from scratch
- Real-time sync: WebSocket + **CRDT** (Yjs/Automerge) → conflict-free merging, offline edits; or OT (server-authoritative)
- Presence: cursors/selections via awareness protocol, throttled
- Persistence: periodic snapshots + update log; autosave status indicator
- Offline: local IndexedDB copy of the doc; sync on reconnect
- Performance: virtualize long documents / render only visible blocks, Web Worker for heavy ops
- Permissions: viewer/commenter/editor enforced server-side

---

# 11. Worked Design 6: Design-System Component (e.g. Data Table / Select)

When asked to design a **component** rather than an app:

1. **API design**: props, composition (compound components), controlled/uncontrolled, events, slots, generics (TS)
2. **States**: default, hover, focus, disabled, loading, error, empty, read-only
3. **Accessibility**: correct ARIA pattern from APG, keyboard interactions table, focus management
4. **Performance**: virtualization for large data, memoization, stable callbacks
5. **Theming/styling**: tokens, variants, className/style overrides, RTL
6. **Testing**: unit (logic), interaction (RTL), visual regression (Storybook/Chromatic), a11y (axe)
7. **Documentation**: Storybook stories, usage guidelines, migration notes
8. **Versioning**: semver, deprecations, codemods

```tsx
<DataTable
  data={rows}
  columns={columns}                  // { id, header, accessor, cell?, sortable?, width? }
  getRowId={(r) => r.id}
  sorting={sorting} onSortingChange={setSorting}           // controlled or uncontrolled
  pagination={{ pageIndex, pageSize }} onPaginationChange={…}
  selection="multiple" onSelectionChange={…}
  loading={isFetching} emptyState={<Empty />}
  virtualized
/>
```

---

# 12. Answer Template (say this out loud)

```text
1. "Let me clarify requirements…" (list functional + non-functional; confirm scope)
2. "High-level architecture…" (draw boxes: UI, state, cache, network, server, CDN)
3. "Rendering strategy is X because…"
4. "Data model on the client…" (entities, normalization, server vs client state)
5. "APIs…" (endpoints, pagination, real-time channel, error format)
6. "Component breakdown…" (tree + key props)
7. "Deep dives: performance, UX states, accessibility, offline, security, observability"
8. "Trade-offs & what I'd do with more time…"
```

---

# 13. Interview Answers (short)

**Cursor vs offset pagination?**

> "Offset (`page=3`) is simple and allows jumping to a page, but items shift when new data is inserted, causing duplicates or gaps, and deep offsets are slow on the database. Cursor pagination returns items after a stable cursor — ideal for infinite feeds and real-time data — but you can't jump to arbitrary pages."

**WebSocket vs SSE vs polling?**

> "Polling is simplest but wasteful and laggy. SSE gives a lightweight one-way server-to-client stream over HTTP with automatic reconnection — great for notifications and feed updates. WebSockets are full-duplex for chat, collaboration and games, but need connection management, heartbeats and reconnection logic, and are harder to scale behind load balancers."

---

# 14. Key Takeaways

- Use **RADIO**: Requirements → Architecture → Data model → Interface → Optimizations.
- Clarify non-functional requirements early (scale, devices, SEO, offline, a11y).
- Choose rendering per route; separate server vs client state; normalize entities.
- Know protocols (REST/GraphQL/SSE/WebSocket) and pagination trade-offs.
- Deep dives: virtualization, code splitting, caching, optimistic UI, offline, a11y, security, observability.
- Always state trade-offs.

## Practice Tracker

- [ ] News feed
- [ ] Autocomplete
- [ ] Chat app
- [ ] E-commerce listing/PDP
- [ ] Collaborative editor
- [ ] Design-system Select/DataTable
- [ ] Bonus: Image gallery (Pinterest), video player (YouTube), notifications system, dashboard with real-time charts, file upload with resumable chunks
