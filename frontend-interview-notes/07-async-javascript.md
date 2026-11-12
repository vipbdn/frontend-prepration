# 07 — Async JavaScript 🔥🔥

> **Interview + Understanding Guide** — `00-starter-roadmap.md` Section 7
>
> Goal: Understand how JavaScript does async work on a single thread, move fluently between callbacks → promises → `async/await`, and avoid the classic sequential/parallel and loop mistakes.

---

# 1. Synchronous vs Asynchronous ⭐⭐⭐

JavaScript runs on a **single thread** with **one call stack** — it can only execute one piece of code at a time.

```js
// Synchronous — each line waits for the previous one
console.log("A");
heavyCalculation(); // blocks everything (UI freezes) until done
console.log("B");
```

**Asynchronous** code starts an operation, **hands it off** to the environment (browser / Node), and continues. When the operation finishes, a **callback is queued** and runs later, when the call stack is empty.

```js
console.log("A");
setTimeout(() => console.log("B"), 0);
console.log("C");
// A, C, B
```

```text
JS thread:  run A ─▶ register timer ─▶ run C ─▶ (stack empty) ─▶ run B
Browser:                 └── timer (Web API) ──▶ queue callback ──┘
```

> JavaScript itself is single-threaded, but the **runtime** (browser/Node) is not — timers, network, file I/O happen outside the JS thread. The **event loop** connects them (details in `08-event-loop.md`).

### Blocking vs non-blocking

| Blocking | Non-blocking |
| --- | --- |
| Long loops, `JSON.parse` on huge data, sync XHR, `alert` | `setTimeout`, `fetch`, event listeners, promises |
| Freezes UI, delays all events | UI stays responsive |

Heavy CPU work stays blocking **even inside a promise** — promises don't create threads. Move it to a **Web Worker** (`24-web-workers.md`).

---

# 2. Callbacks ⭐⭐⭐

A **callback** is a function passed to another function to be called later.

```js
// Synchronous callback
[1, 2, 3].map((n) => n * 2);

// Asynchronous callback
setTimeout(() => console.log("later"), 1000);
button.addEventListener("click", () => console.log("clicked"));
```

### Error-first callbacks (Node convention)

```js
fs.readFile("a.txt", "utf8", (err, data) => {
  if (err) return console.error(err);
  console.log(data);
});
```

### Callback Hell (Pyramid of Doom)

```js
getUser(1, (err, user) => {
  if (err) return handle(err);
  getOrders(user.id, (err, orders) => {
    if (err) return handle(err);
    getDetails(orders[0].id, (err, details) => {
      if (err) return handle(err);
      render(details);
    });
  });
});
```

Problems:
- Hard to read, deeply nested
- Error handling repeated at every level
- **Inversion of control** — the library may call your callback twice, never, or synchronously
- Hard to run things in parallel or compose

Solution path: **Promises** (`06-promises.md`) → **`async/await`**.

---

# 3. `async` / `await` ⭐⭐⭐

`async/await` is **syntax on top of promises** that lets async code read like sync code.

```js
async function loadDashboard() {
  const user = await getUser(1);
  const orders = await getOrders(user.id);
  return orders.length;
}
```

### Rules

1. An `async` function **always returns a promise**.
   - `return value` → promise fulfilled with value
   - `throw err` → promise rejected with err
   - `return somePromise` → follows that promise
2. `await` **pauses the async function** (not the thread!) until the promise settles, then:
   - fulfilled → evaluates to the value
   - rejected → **throws** the reason at that line
3. Code after `await` runs as a **microtask** — even `await 5` (non-promise) yields.
4. `await` works only inside `async` functions — or at the **top level of ES modules**.

```js
async function f() {
  return 1;
}
f();                 // Promise {<fulfilled>: 1}
f().then(console.log); // 1

async function g() {
  throw new Error("x");
}
g().catch((e) => console.log(e.message)); // "x"
```

### `await` doesn't block the thread

```js
async function task() {
  console.log("1");
  await null;
  console.log("3");
}

task();
console.log("2");
// 1, 2, 3
```

Everything before the first `await` runs **synchronously**. At `await`, the function suspends and control returns to the caller.

---

# 4. Error Handling with `try/catch` ⭐⭐⭐

```js
async function loadUser(id) {
  try {
    const res = await fetch(`/api/users/${id}`);
    if (!res.ok) throw new Error(`HTTP ${res.status}`); // fetch doesn't reject on 404/500
    return await res.json();
  } catch (err) {
    console.error("Failed:", err);
    throw err;           // re-throw if callers need to know
  } finally {
    setLoading(false);   // always runs
  }
}
```

### `return` vs `return await` inside try

```js
async function a() {
  try {
    return fetchData();       // ❗ rejection NOT caught here — promise returned as-is
  } catch {
    return "fallback";
  }
}

async function b() {
  try {
    return await fetchData(); // ✅ rejection thrown here, caught below
  } catch {
    return "fallback";
  }
}
```

> Outside `try/catch`, `return await` is unnecessary. **Inside** `try`, you need it.

### Alternative: catch on the call

```js
const user = await getUser(id).catch(() => null);
```

### Go-style helper

```js
const to = (promise) => promise.then((data) => [null, data]).catch((err) => [err, null]);

const [err, user] = await to(getUser(1));
if (err) { /* handle */ }
```

---

# 5. Sequential vs Parallel `await` ⭐⭐⭐

The most common real-world async mistake: accidentally making independent work **sequential**.

```js
// ❌ Sequential — 2s + 2s = 4s (second request waits for the first)
const user = await fetchUser();     // 2s
const posts = await fetchPosts();   // 2s

// ✅ Parallel — ~2s (both start immediately)
const [user, posts] = await Promise.all([fetchUser(), fetchPosts()]);

// ✅ Also parallel — start first, await later
const userP = fetchUser();
const postsP = fetchPosts();
const user2 = await userP;
const posts2 = await postsP;
```

⚠️ The "start first, await later" form has a subtle issue: if `postsP` rejects while you're still awaiting `userP`, it's briefly an **unhandled rejection**. `Promise.all` avoids this — prefer it.

Use **sequential** only when the second call **needs** the first result:

```js
const user = await fetchUser();
const orders = await fetchOrders(user.id); // depends on user
```

---

# 6. Loops + `await` ⭐⭐⭐

### `forEach` does NOT wait ❌

```js
async function processAll(ids) {
  ids.forEach(async (id) => {
    await save(id);
  });
  console.log("done"); // runs immediately — saves still pending
}
```

`forEach` ignores the promises returned by its callback.

### `for...of` — sequential ✅

```js
for (const id of ids) {
  await save(id); // one after another
}
console.log("done"); // after all saves
```

### `map` + `Promise.all` — parallel ✅

```js
await Promise.all(ids.map((id) => save(id)));
console.log("done");
```

### Parallel with a limit ✅

```js
await promisePool(ids.map((id) => () => save(id)), 3); // see 06-promises.md §6.5
```

| Loop | Behaviour with `await` |
| --- | --- |
| `for`, `for...of`, `while` | ✅ Sequential, waits |
| `forEach`, `map`, `filter`, `reduce` | ❌ Don't wait (map returns promises you can `Promise.all`) |
| `for await...of` | ✅ Iterates async iterables / arrays of promises sequentially |

### `filter` with async predicate ❌

```js
const active = users.filter(async (u) => await isActive(u)); // ❌ promise is truthy → keeps all

// ✅
const flags = await Promise.all(users.map((u) => isActive(u)));
const active2 = users.filter((_, i) => flags[i]);
```

---

# 7. Async Iteration — `for await...of`

Iterates over **async iterables** (or iterables of promises), awaiting each value.

```js
const promises = [fetchA(), fetchB(), fetchC()];

for await (const result of promises) {
  console.log(result); // in order, each awaited
}
```

### Async generators

```js
async function* paginate(url) {
  let next = url;
  while (next) {
    const res = await fetch(next);
    const { items, nextPage } = await res.json();
    yield items;           // hand one page to the consumer
    next = nextPage;
  }
}

for await (const page of paginate("/api/items?page=1")) {
  render(page);
  if (enough()) break;    // stops fetching further pages
}
```

Real-world uses: paginated APIs, streaming responses (`response.body` is an async iterable in modern browsers/Node), reading files line by line in Node.

More in `18-iterators-and-generators.md`.

---

# 8. Top-Level `await`

Allowed in **ES modules** (not classic scripts or CommonJS):

```js
// config.mjs
const res = await fetch("/config.json");
export const config = await res.json();
```

⚠️ Any module importing this **waits** until it finishes — can delay app startup.

---

# 9. Converting Between Styles

```js
// Callback → Promise
const wait = (ms) => new Promise((resolve) => setTimeout(resolve, ms));

// Promise chain → async/await
function getCount() {
  return fetchUser()
    .then((u) => fetchPosts(u.id))
    .then((posts) => posts.length)
    .catch(() => 0);
}

async function getCount2() {
  try {
    const u = await fetchUser();
    const posts = await fetchPosts(u.id);
    return posts.length;
  } catch {
    return 0;
  }
}
```

### Promises vs async/await — when to use which

| `async/await` | `.then` chains |
| --- | --- |
| Most code — readable, try/catch, easy debugging (stack traces, breakpoints) | Quick one-liners, transformations, `Promise.all(...).then(...)` |
| Easy to use earlier results in later steps | Fine for simple linear pipelines |

They're interchangeable — `async/await` **is** promises.

---

# 10. Output Questions 🧪

### Q1

```js
async function f() {
  console.log("a");
  await null;
  console.log("b");
}
console.log("start");
f();
console.log("end");
```

<details><summary>Answer</summary>

`start a end b`

</details>

### Q2

```js
async function f() {
  return 10;
}
console.log(f());
f().then(console.log);
```

<details><summary>Answer</summary>

`Promise { 10 }` then `10`.

</details>

### Q3

```js
async function f() {
  try {
    return Promise.reject("x");
  } catch {
    return "caught";
  }
}
f().then(console.log, (e) => console.log("rejected:", e));
```

<details><summary>Answer</summary>

`rejected: x` — without `await`, the rejection isn't thrown inside `try`.

</details>

### Q4

```js
const delay = (ms, v) => new Promise((r) => setTimeout(() => r(v), ms));

async function run() {
  console.time("t");
  const a = await delay(100, "a");
  const b = await delay(100, "b");
  console.timeEnd("t");
}
run();
```

<details><summary>Answer</summary>

~`200ms` — sequential. With `Promise.all`, ~`100ms`.

</details>

### Q5

```js
async function run() {
  [1, 2, 3].forEach(async (n) => {
    await null;
    console.log(n);
  });
  console.log("done");
}
run();
```

<details><summary>Answer</summary>

`done 1 2 3` — `forEach` doesn't wait.

</details>

### Q6

```js
async function a() {
  await b();
  console.log("a");
}
async function b() {
  throw new Error("b failed");
}
a().catch((e) => console.log(e.message));
```

<details><summary>Answer</summary>

`b failed` — `await` re-throws the rejection, `a` rejects, `"a"` is never logged.

</details>

---

# 11. Interview Answers

**How does async work if JS is single-threaded?**

> "JavaScript has one call stack, but the runtime provides APIs like timers and network that run outside it. When those finish, their callbacks are queued, and the event loop pushes them onto the stack when it's empty. Promise callbacks go into the microtask queue, which runs before timers. So JS doesn't run things in parallel — it just doesn't wait idly for I/O."

**What does `async/await` do?**

> "It's syntax over promises. An async function always returns a promise. `await` pauses only that function until the promise settles — returning the value or throwing the error — while the rest of the program keeps running. The code after `await` resumes as a microtask."

**Common async mistakes?**

> "Awaiting independent calls sequentially instead of using `Promise.all`; using `forEach` with async callbacks; returning without `await` inside `try/catch`; forgetting that `fetch` doesn't reject on HTTP errors; and not handling rejections."

---

# 12. Key Takeaways

- JS is single-threaded; async work is offloaded and callbacks are queued.
- Callbacks → callback hell + inversion of control → promises → `async/await`.
- `async` always returns a promise; `await` suspends the function, not the thread.
- Code before the first `await` is synchronous; after it is a microtask.
- `try/catch` + `return await` inside `try`.
- Independent work → `Promise.all`. Dependent work → sequential.
- `forEach` doesn't await; use `for...of` (sequential) or `map + Promise.all` (parallel).
- `for await...of` + async generators for streams and pagination.

## Readiness Checklist

- [ ] Explain sync vs async with a timeline diagram
- [ ] Rewrite callback hell → promises → async/await
- [ ] Explain all 4 rules of `async/await`
- [ ] Spot sequential-vs-parallel mistakes in code
- [ ] Explain why `forEach` + `await` fails and give 2 fixes
- [ ] Write an async generator for pagination
