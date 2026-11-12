# 23 — JavaScript Performance 🔥🔥

> **Interview + Understanding Guide** — `00-starter-roadmap.md` Section 23
>
> Goal: Understand how the browser renders, what makes pages slow (reflow, repaint, layout thrashing, long tasks, big bundles), and the toolbox to fix it — debounce/throttle, memoization, lazy loading, code splitting, tree shaking, virtualization, `requestAnimationFrame`, `requestIdleCallback`, and DevTools.

---

# 1. How the Browser Renders a Frame

```text
JavaScript → Style → Layout → Paint → Composite
   (JS)     (recalc   (reflow:   (fill    (layer
             styles)   geometry)  pixels)  assembly, GPU)
```

| Step | What happens |
| --- | --- |
| **Style** | Compute which CSS rules apply to each element |
| **Layout (reflow)** | Compute size & position of every affected element |
| **Paint (repaint)** | Fill in pixels: text, colors, borders, shadows, images |
| **Composite** | Combine layers on the GPU and show on screen |

At 60 Hz, each frame has **~16.7 ms** — and the browser needs part of that, so JS should take **< ~10 ms per frame** to stay smooth. A task longer than **50 ms** is a **long task** (hurts responsiveness).

---

# 2. Reflow vs Repaint ⭐

| | Reflow (Layout) | Repaint |
| --- | --- | --- |
| Triggered by | Geometry changes: `width`, `height`, `margin`, `padding`, `top/left`, font size, adding/removing elements, content changes, window resize | Visual-only changes: `color`, `background`, `visibility`, `box-shadow`, `outline` |
| Cost | **Expensive** — can cascade to parents, children, siblings | Cheaper, but still work |
| Implies | Always followed by repaint | Doesn't require reflow |

### Cheapest: compositor-only properties

`transform` and `opacity` can often skip layout **and** paint — only compositing (GPU).

```css
/* ❌ animates layout every frame */
.box { transition: left 0.3s; }
.box.move { left: 200px; }

/* ✅ compositor-only */
.box { transition: transform 0.3s; }
.box.move { transform: translateX(200px); }
```

`will-change: transform` hints the browser to promote an element to its own layer (use sparingly — layers cost memory).

---

# 3. Layout Thrashing ⭐

The browser **batches** style changes and recalculates layout lazily. But **reading** a layout property (`offsetHeight`, `getBoundingClientRect()`, `scrollTop`, `clientWidth`, `getComputedStyle`) **forces a synchronous layout** if there are pending changes.

Interleaving **write → read → write → read** in a loop = **forced synchronous layouts** every iteration = **layout thrashing**.

```js
// ❌ Thrashing: read (forces layout) + write, N times
for (const box of boxes) {
  const width = container.offsetWidth; // READ → forced layout
  box.style.width = width / 2 + "px";  // WRITE → invalidates layout
}

// ✅ Batch: read once, then write
const width = container.offsetWidth;    // one layout
for (const box of boxes) {
  box.style.width = width / 2 + "px";   // writes only
}
```

### Rule: **batch reads, then batch writes**

```js
// Read phase
const heights = items.map((el) => el.offsetHeight);
// Write phase
items.forEach((el, i) => (el.style.height = heights[i] * 2 + "px"));
```

For complex cases, schedule writes in `requestAnimationFrame` (libraries like **fastdom** do this).

### Other DOM performance tips

- Insert many nodes at once with `DocumentFragment` or a single `innerHTML`/`append(...nodes)`.
- Change **classes** instead of many individual inline styles.
- Avoid animating layout properties; use `transform`/`opacity`.
- Hide → modify → show (or modify detached elements) for big batch updates.
- CSS `contain: layout paint` / `content-visibility: auto` to limit the scope of layout/paint work.

---

# 4. Debouncing, Throttling, Memoization (recap)

| Technique | Fixes | Details |
| --- | --- | --- |
| **Debounce** | Too many calls during bursts (typing → API) | `09-debounce-and-throttle.md` |
| **Throttle** | Too-frequent handlers (scroll, resize, mousemove) | `09-debounce-and-throttle.md` |
| **Memoization** | Recomputing expensive pure results / unnecessary React re-renders | `10-memoization-and-caching.md` |
| **Passive listeners** | Scroll jank from touch/wheel listeners | `{ passive: true }` |
| **IntersectionObserver** | Scroll handlers that check visibility | `12-dom-and-browser-apis.md` |

---

# 5. `requestAnimationFrame` ⭐

Schedules a callback **right before the next paint** — synchronized with the display refresh rate. Use it for **visual updates and animations**.

```js
function animate(element) {
  let start;
  function frame(timestamp) {
    start ??= timestamp;
    const progress = Math.min((timestamp - start) / 1000, 1); // 1s animation
    element.style.transform = `translateX(${progress * 300}px)`;
    if (progress < 1) requestAnimationFrame(frame);
  }
  requestAnimationFrame(frame);
}
```

Why not `setTimeout(fn, 16)`?
- Timers aren't synced to frames → dropped/doubled frames (jank)
- rAF **pauses** in background tabs → saves CPU/battery
- rAF batches with the browser's rendering work

Cancel with `cancelAnimationFrame(id)`.

---

# 6. `requestIdleCallback`

Runs low-priority work when the browser is **idle** (between frames, nothing urgent).

```js
const queue = [...analyticsEvents];

function processIdle(deadline) {
  while (deadline.timeRemaining() > 0 && queue.length) {
    send(queue.shift());
  }
  if (queue.length) requestIdleCallback(processIdle);
}

requestIdleCallback(processIdle, { timeout: 2000 }); // run within 2s even if never idle
```

Use for: analytics, prefetching, non-urgent indexing, logging. Not for anything visual or DOM-heavy. (Not supported in all browsers historically — Safari added it late; feature-detect.)

---

# 7. Breaking Up Long Tasks

The main thread does **everything**: JS, layout, paint, input handling. A 500 ms loop = 500 ms of frozen UI.

```js
// ❌ One long task
processAll(hugeArray);

// ✅ Yield to the browser between chunks
async function processInChunks(items, chunkSize = 200) {
  for (let i = 0; i < items.length; i += chunkSize) {
    items.slice(i, i + chunkSize).forEach(processItem);
    await new Promise((r) => setTimeout(r, 0)); // or scheduler.yield() where supported
  }
}
```

For CPU-heavy work (parsing, image processing, crypto), move it **off the main thread** → **Web Workers** (`24-web-workers.md`).

---

# 8. Lazy Loading ⭐

Load things **only when needed**.

```html
<!-- Native image/iframe lazy loading -->
<img src="photo.jpg" loading="lazy" width="400" height="300" alt="..." />
<iframe src="video.html" loading="lazy"></iframe>
```

- Always set `width`/`height` (or `aspect-ratio`) → prevents **layout shift (CLS)**.
- Don't lazy-load above-the-fold images (especially the LCP image) — use `fetchpriority="high"` instead.

```js
// Lazy-load a module on interaction
button.addEventListener("click", async () => {
  const { openChat } = await import("./chat-widget.js");
  openChat();
});
```

```jsx
// React
const Settings = React.lazy(() => import("./Settings"));
<Suspense fallback={<Spinner />}><Settings /></Suspense>
```

---

# 9. Code Splitting & Tree Shaking ⭐

| | Code splitting | Tree shaking |
| --- | --- | --- |
| Goal | Smaller **initial** bundle | Smaller **total** bundle |
| How | Dynamic `import()` → separate chunks loaded on demand | Remove unused exports (static ESM analysis) |
| Example | Route-based chunks, lazy modals/charts | `import { debounce } from "lodash-es"` instead of all of lodash |

Details in `16-javascript-modules.md`.

### Bundle hygiene

- Analyze: `webpack-bundle-analyzer`, `rollup-plugin-visualizer`, `source-map-explorer`.
- Replace heavy libraries (moment → date-fns/dayjs/`Intl`).
- Import only what you use; prefer ESM packages.
- Compression (Brotli/gzip), minification, long-term caching with content hashes.

---

# 10. Virtualization (Windowing) ⭐

Rendering 10,000 rows = 10,000+ DOM nodes → slow initial render, heavy memory, janky scroll.

**Virtualization** renders **only the rows visible in the viewport** (plus a small buffer), reusing DOM as you scroll.

```text
Total list: 10,000 items (height = 10,000 × 40px = 400,000px)
Viewport:   600px → ~15 visible rows
Rendered:   ~25 rows (15 + buffer) inside a 400,000px-tall spacer
```

### Core idea (simplified)

```jsx
function VirtualList({ items, rowHeight = 40, height = 600, overscan = 5 }) {
  const [scrollTop, setScrollTop] = useState(0);

  const start = Math.max(0, Math.floor(scrollTop / rowHeight) - overscan);
  const visibleCount = Math.ceil(height / rowHeight) + overscan * 2;
  const end = Math.min(items.length, start + visibleCount);

  return (
    <div style={{ height, overflowY: "auto" }} onScroll={(e) => setScrollTop(e.currentTarget.scrollTop)}>
      <div style={{ height: items.length * rowHeight, position: "relative" }}>
        {items.slice(start, end).map((item, i) => (
          <div
            key={item.id}
            style={{ position: "absolute", top: (start + i) * rowHeight, height: rowHeight, width: "100%" }}
          >
            {item.label}
          </div>
        ))}
      </div>
    </div>
  );
}
```

Libraries: **TanStack Virtual**, **react-window**, **react-virtuoso** (dynamic heights). CSS alternative for simpler cases: `content-visibility: auto`.

Trade-offs: Ctrl+F browser search can't find unrendered rows; accessibility needs care; dynamic heights are harder.

---

# 11. Performance API

Measure in code (real-user monitoring, custom timings):

```js
performance.mark("filter-start");
filterProducts();
performance.mark("filter-end");
performance.measure("filter", "filter-start", "filter-end");
console.log(performance.getEntriesByName("filter")[0].duration);

const t0 = performance.now(); // high-resolution timestamp (ms, fractional)
doWork();
console.log(`took ${performance.now() - t0}ms`);

// Observe long tasks / paint / layout shifts
new PerformanceObserver((list) => {
  for (const entry of list.getEntries()) console.log(entry.entryType, entry.duration);
}).observe({ type: "longtask", buffered: true });

// Navigation timing (TTFB etc.)
const [nav] = performance.getEntriesByType("navigation");
console.log("TTFB", nav.responseStart - nav.requestStart);
```

### Core Web Vitals (know these names)

| Metric | Measures | Good |
| --- | --- | --- |
| **LCP** — Largest Contentful Paint | Loading: when main content appears | ≤ 2.5 s |
| **INP** — Interaction to Next Paint | Responsiveness to input (replaced FID in 2024) | ≤ 200 ms |
| **CLS** — Cumulative Layout Shift | Visual stability | ≤ 0.1 |

Library: `web-vitals`. Fixes: optimize images/fonts & server time (LCP), break up long tasks & reduce JS (INP), reserve space for images/ads/embeds (CLS).

---

# 12. Chrome DevTools

### Performance panel

1. Record while reproducing the slow interaction (enable CPU throttling to simulate mobile).
2. Look for **red triangles / long tasks** in the Main track.
3. The **flame chart** shows which functions took time; purple = layout/style, green = paint.
4. "Forced reflow" warnings → layout thrashing.
5. Bottom-up / Call tree tabs to find hot functions.

### Memory panel

Heap snapshots, allocation timeline — see `14-memory-management.md`.

### Other tools

- **Lighthouse** — audit with scores + suggestions
- **Network panel** — waterfall, sizes, caching, throttling
- **Coverage** tab — unused JS/CSS per file
- **Rendering** tab — paint flashing, layout shift regions, FPS meter
- **React DevTools Profiler** — which components rendered, why, and how long

---

# 13. React Performance Checklist

- Keep state **as low as possible** (local); avoid putting fast-changing state in context at the top.
- Split context; memoize provider values.
- `React.memo` + stable props (`useCallback`/`useMemo`) for expensive children.
- Stable, unique `key`s (not array index for dynamic lists).
- Virtualize long lists.
- Lazy-load routes and heavy components.
- Debounce inputs that trigger expensive work; `useDeferredValue` / `useTransition` for non-urgent rendering.
- Avoid creating heavy objects in render; avoid unnecessary effects (derive state instead).
- Measure with the Profiler before optimizing.

---

# 14. Interview Answers

**Reflow vs repaint**

> "Reflow — or layout — recalculates the position and size of elements when geometry changes, like width, margin or adding elements. It's expensive and can cascade. Repaint redraws pixels when only visual properties change, like color. Animating `transform` and `opacity` is cheapest because it can skip both and only composite."

**What is layout thrashing?**

> "It's when JavaScript alternates between writing styles and reading layout properties like `offsetHeight` in a loop. Each read forces the browser to recalculate layout synchronously. The fix is to batch all reads first, then all writes, or schedule writes in `requestAnimationFrame`."

**How would you speed up a slow page?**

> "Measure first — Lighthouse, Web Vitals, the Performance panel. For load: code-split routes, tree-shake, lazy-load images and non-critical components, compress and cache assets. For runtime: break up long tasks or move them to a Web Worker, debounce/throttle frequent handlers, avoid layout thrashing, virtualize long lists, and in React reduce unnecessary re-renders with better state placement and memoization."

---

# 15. Key Takeaways

- Pipeline: JS → Style → Layout → Paint → Composite; ~16 ms per frame; long task > 50 ms.
- Reflow (geometry) > repaint (visuals) > composite-only (`transform`, `opacity`).
- Layout thrashing: interleaved reads/writes → batch reads, then writes.
- rAF for visual work; rIC for idle work; chunk or offload long tasks.
- Lazy load, code split, tree shake, virtualize.
- Measure: Performance API, Web Vitals (LCP, INP, CLS), DevTools Performance/Memory, React Profiler.

## Readiness Checklist

- [ ] Draw the rendering pipeline
- [ ] List properties that trigger reflow vs repaint vs composite
- [ ] Fix a layout-thrashing loop
- [ ] Explain virtualization and sketch the math
- [ ] Use `performance.mark/measure`
- [ ] Explain LCP, INP, CLS and one fix for each
