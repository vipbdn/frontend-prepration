# 24 — Web Workers & Concurrency

> **Interview + Understanding Guide** — `00-starter-roadmap.md` Section 24
>
> Goal: Understand why the main thread matters, how Web Workers run JavaScript in parallel, how they communicate (structured cloning, transferables, `SharedArrayBuffer`), and when they're the right tool.

---

# 1. The Main Thread

In the browser, the **main thread** does almost everything:

```text
Main thread:  JS execution · Style · Layout · Paint · Input handling · Timers · Network callbacks
```

If your JS runs for 2 seconds, **nothing else happens** for 2 seconds: no clicks, no scrolling, no animations — the page freezes.

```js
button.onclick = () => {
  const result = fibonacci(45); // ~seconds of CPU work → UI frozen
  output.textContent = result;
};
```

Async (`setTimeout`, promises) **doesn't help** for CPU-heavy work — it only postpones it; it still runs on the same thread.

---

# 2. What is a Web Worker? ⭐

A **Web Worker** runs a JavaScript file on a **separate background thread**, in parallel with the main thread.

```text
Main thread  ──postMessage──▶  Worker thread
   (UI)      ◀──postMessage──   (heavy computation)
```

### Why Web Workers are needed

- Keep the UI **responsive** during CPU-heavy work
- Use **multiple CPU cores**
- Isolate heavy or untrusted computation

### What workers CAN'T do

| ❌ Not available in workers | ✅ Available in workers |
| --- | --- |
| DOM (`document`, `window`, elements) | `fetch`, `XMLHttpRequest`, WebSockets |
| `localStorage` / `sessionStorage` | `IndexedDB`, Cache API |
| `alert`, direct UI access | Timers (`setTimeout`, `setInterval`) |
| Parent's variables/closures | `importScripts()` / ES module imports |
| | `crypto`, `OffscreenCanvas`, `WebAssembly` |
| | `self`, `postMessage`, `onmessage` |

Workers have their **own global scope** (`self`), own event loop, own memory. They share **nothing** with the main thread by default.

---

# 3. Basic Usage ⭐

### Main thread

```js
// main.js
const worker = new Worker(new URL("./worker.js", import.meta.url), { type: "module" });
// (classic: new Worker("worker.js"))

worker.postMessage({ type: "fib", n: 42 }); // send data

worker.onmessage = (event) => {
  console.log("Result:", event.data);        // receive data
};

worker.onerror = (event) => {
  console.error("Worker error:", event.message);
};

// Later, when done
worker.terminate(); // kill immediately
```

### Worker thread

```js
// worker.js
function fib(n) {
  return n < 2 ? n : fib(n - 1) + fib(n - 2);
}

self.onmessage = (event) => {
  const { type, n } = event.data;
  if (type === "fib") {
    const result = fib(n);
    self.postMessage(result); // send back
  }
};

// self.close(); // worker can stop itself
```

> With bundlers (Vite/webpack 5), `new Worker(new URL("./worker.js", import.meta.url))` is the recognized pattern; they bundle the worker file automatically.

---

# 4. Worker Communication ⭐

- `postMessage(data)` → sends a message
- `onmessage` / `addEventListener("message")` → receives it (`event.data`)
- Messages are **asynchronous** and arrive as **tasks** in the receiver's event loop.

### Request/response with IDs (promise wrapper)

Messages are just events — to match responses to requests, add IDs:

```js
// main.js
function createWorkerClient(url) {
  const worker = new Worker(url, { type: "module" });
  const pending = new Map();
  let nextId = 0;

  worker.onmessage = ({ data }) => {
    const { id, result, error } = data;
    const p = pending.get(id);
    if (!p) return;
    pending.delete(id);
    error ? p.reject(new Error(error)) : p.resolve(result);
  };

  return {
    run(payload) {
      const id = nextId++;
      return new Promise((resolve, reject) => {
        pending.set(id, { resolve, reject });
        worker.postMessage({ id, payload });
      });
    },
    terminate: () => worker.terminate(),
  };
}

// worker.js
self.onmessage = ({ data: { id, payload } }) => {
  try {
    self.postMessage({ id, result: heavyWork(payload) });
  } catch (err) {
    self.postMessage({ id, error: err.message });
  }
};

// usage
const client = createWorkerClient(new URL("./worker.js", import.meta.url));
const result = await client.run({ rows: bigCsvText });
```

Libraries like **Comlink** do this for you (call worker functions as if they were local async functions).

---

# 5. Structured Cloning ⭐

`postMessage` **copies** data using the **structured clone algorithm** (same as `structuredClone()`).

```js
worker.postMessage({ date: new Date(), map: new Map([[1, "a"]]), nested: { arr: [1, 2] } });
// Worker receives a deep copy — changes there don't affect the main thread's object
```

| ✅ Clonable | ❌ Not clonable (throws `DataCloneError`) |
| --- | --- |
| Primitives (except Symbol), objects, arrays | Functions |
| Date, RegExp, Map, Set | DOM nodes |
| ArrayBuffer, TypedArrays, Blob, File, ImageData | Symbols |
| Error objects | Class instances keep data but **lose their prototype/methods** |
| Circular references ✅ | Getters/setters, property descriptors |

⚠️ Cloning **large** data (e.g., a 100 MB buffer) costs time and **doubles memory**.

---

# 6. Transferable Objects ⭐

Instead of copying, you can **transfer ownership** of certain objects — zero-copy, near-instant. The sender **loses access**.

```js
const buffer = new ArrayBuffer(100 * 1024 * 1024); // 100 MB
console.log(buffer.byteLength); // 104857600

worker.postMessage(buffer, [buffer]); // 2nd arg: list of objects to transfer

console.log(buffer.byteLength); // 0 — "detached"; main thread can't use it anymore
```

Transferables: `ArrayBuffer`, `MessagePort`, `ImageBitmap`, `OffscreenCanvas`, `ReadableStream`/`WritableStream`, `VideoFrame`…

```js
// Transfer image pixels to a worker for processing, and back
const imageData = ctx.getImageData(0, 0, w, h);
worker.postMessage(imageData, [imageData.data.buffer]);
```

| Structured clone | Transfer |
| --- | --- |
| Copy — both sides have data | Move — only receiver has data |
| Cost grows with size | ~Constant time |
| Works for most data | Only transferable types |

---

# 7. `SharedArrayBuffer` — Basic Understanding

A `SharedArrayBuffer` is memory **shared** between threads — both can read/write the **same** bytes simultaneously (true shared memory).

```js
const shared = new SharedArrayBuffer(4); // 4 bytes
const view = new Int32Array(shared);
worker.postMessage(shared); // NOT copied — same memory

// Use Atomics to avoid race conditions
Atomics.add(view, 0, 1);
Atomics.load(view, 0);
Atomics.wait / Atomics.notify // synchronization
```

Key points:
- Shared memory introduces **race conditions** → use `Atomics` for safe operations.
- Requires **cross-origin isolation** (security after Spectre): serve with
  `Cross-Origin-Opener-Policy: same-origin` and `Cross-Origin-Embedder-Policy: require-corp`.
- Used by heavy apps (WebAssembly threads, video/audio processing, games, Figma-like editors).

For most frontend work: **know it exists and why it needs isolation**.

---

# 8. Types of Workers

| Type | Purpose |
| --- | --- |
| **Dedicated Worker** (`new Worker`) | Belongs to one page; background computation (this file's focus) |
| **Shared Worker** (`new SharedWorker`) | Shared by multiple tabs/iframes of the same origin (e.g., one WebSocket for all tabs); limited support on some mobile browsers |
| **Service Worker** | Network proxy between app and network: offline caching, push notifications, background sync — PWAs. Event-driven, no DOM, lifecycle independent of pages |
| **Worklets** (Audio, Paint) | Tiny specialized hooks into rendering/audio pipelines |

### Web Worker vs Service Worker (common question)

| Web Worker | Service Worker |
| --- | --- |
| Offload CPU work from the main thread | Intercept network requests, cache, offline, push |
| Lives as long as the page (or until terminated) | Lives independently, started on events, shared across pages of its scope |
| Created by page script | Registered (`navigator.serviceWorker.register`) |
| Communicates via `postMessage` | Communicates via `postMessage`, intercepts `fetch` events |

---

# 9. When to Use Web Workers ⭐

### ✅ Good fits

- Parsing / transforming **large data** (CSV/JSON with 100k+ rows, sorting, filtering, aggregations)
- **Image/video/audio** processing (filters, compression, resizing)
- **Cryptography**, hashing, compression (e.g., zipping files client-side)
- Syntax highlighting / linting in code editors, search indexing (full-text search)
- Heavy math: charts with huge datasets, simulations, ML inference (TensorFlow.js)
- `OffscreenCanvas` rendering

### ❌ Not worth it

- Small/quick tasks — the cost of starting a worker (~tens of ms) and copying data outweighs gains
- DOM-heavy work — workers can't touch the DOM
- I/O-bound work (`fetch`) — already async and non-blocking on the main thread
- When the bottleneck is rendering/layout, not JS

> Rule: if a task blocks the main thread for **more than ~50 ms** and doesn't need the DOM, consider a worker.

---

# 10. Web Worker in React

```jsx
function useWorker(createWorker) {
  const workerRef = useRef(null);

  useEffect(() => {
    workerRef.current = createWorker();
    return () => workerRef.current.terminate(); // cleanup!
  }, []);

  const run = useCallback(
    (payload) =>
      new Promise((resolve, reject) => {
        const worker = workerRef.current;
        worker.onmessage = (e) => resolve(e.data);
        worker.onerror = (e) => reject(e);
        worker.postMessage(payload);
      }),
    [],
  );

  return run;
}

function ReportPage({ rawCsv }) {
  const run = useWorker(() => new Worker(new URL("./csv.worker.js", import.meta.url), { type: "module" }));
  const [rows, setRows] = useState(null);

  useEffect(() => {
    run(rawCsv).then(setRows);
  }, [rawCsv, run]);

  return rows ? <Table rows={rows} /> : <Spinner />;
}
```

(This simple hook handles one request at a time; use the ID-based client from §4 or Comlink for concurrent calls.)

---

# 11. Interview Answers

**What are Web Workers and why use them?**

> "Web Workers run JavaScript on a background thread, separate from the main thread that handles rendering and user input. Long CPU-heavy tasks on the main thread freeze the UI, and making them async doesn't help because they still run on the same thread. Moving them to a worker keeps the page responsive. Workers can't access the DOM; they communicate with the page through `postMessage`."

**How is data passed to a worker?**

> "By default `postMessage` deep-copies data with the structured clone algorithm — it supports objects, Maps, Dates and circular references but not functions or DOM nodes. For large binary data like `ArrayBuffer`s, you can transfer ownership instead, which is zero-copy but leaves the sender's buffer detached. `SharedArrayBuffer` allows true shared memory with `Atomics`, but requires cross-origin isolation."

---

# 12. Key Takeaways

- Main thread = JS + rendering + input; long JS freezes the UI.
- Async ≠ parallel. Workers = real parallelism on another thread.
- No DOM in workers; communicate with `postMessage`/`onmessage`.
- Data is structured-cloned (copied); transfer `ArrayBuffer`s for zero-copy.
- `SharedArrayBuffer` + `Atomics` for shared memory (needs COOP/COEP).
- Dedicated vs Shared vs Service workers — different purposes.
- Use for CPU-heavy, DOM-free tasks > ~50 ms; always `terminate()` when done.

## Readiness Checklist

- [ ] Explain why async doesn't fix CPU-heavy work
- [ ] Write a worker with `postMessage` both ways
- [ ] Build a promise-based worker client with request IDs
- [ ] Explain structured clone vs transfer vs shared memory
- [ ] Compare Web Worker vs Service Worker
- [ ] List 3 good and 3 bad use cases
