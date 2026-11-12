# 06 — Promises 🔥🔥🔥

> **Interview + Understanding Guide** — `00-starter-roadmap.md` Section 6
>
> Goal: Understand exactly how promises settle, chain and propagate errors; know every static method; implement `Promise.all/race/any/allSettled`, retry, timeout, a concurrency limiter, a queue, and a custom Promise.

---

# 1. What is a Promise?

A **Promise** is an object representing the **eventual result** of an asynchronous operation — a placeholder for a value that isn't available yet.

### Interview Definition

> **A Promise is an object that represents the eventual completion or failure of an asynchronous operation. It starts as pending and settles exactly once — either fulfilled with a value or rejected with a reason. You react to it with `.then`, `.catch` and `.finally`, whose callbacks always run asynchronously as microtasks.**

### Why promises exist

Callbacks led to:
- **Callback hell** (deep nesting)
- **Inversion of control** — you hand your callback to someone else's code and trust them to call it once, with the right args, and handle errors
- **No standard error handling**

Promises give: **chaining**, **one error channel** (`.catch`), a **guarantee** of being settled once, and composition (`Promise.all`, etc.).

---

# 2. States ⭐⭐⭐

```text
                ┌── resolve(value) ──▶  FULFILLED (value)
   PENDING ─────┤
                └── reject(reason) ──▶  REJECTED (reason)

   fulfilled + rejected = "settled"   (final, never changes again)
```

```js
const p = new Promise((resolve, reject) => {
  // executor runs IMMEDIATELY and SYNCHRONOUSLY
  setTimeout(() => resolve("data"), 1000);
});
```

### Rules that get tested

1. The **executor runs synchronously** when the promise is created.
2. A promise settles **only once** — later `resolve`/`reject` calls are ignored.
3. A **throw inside the executor** = `reject(error)`.
4. `.then/.catch/.finally` callbacks run **asynchronously** (microtask), even if the promise is already settled.
5. "Resolved" ≠ "fulfilled": `resolve(anotherPromise)` makes this promise **follow** the other one — it might still end up rejected.

```js
new Promise((resolve, reject) => {
  resolve(1);
  reject(new Error("ignored"));
  resolve(2); // ignored
}).then(console.log); // 1
```

---

# 3. Consuming: `.then`, `.catch`, `.finally` ⭐⭐⭐

```js
promise
  .then(onFulfilled, onRejected?) // both optional
  .catch(onRejected)              // = .then(undefined, onRejected)
  .finally(onFinally);            // runs either way, no arguments
```

### Every `.then` returns a NEW promise

What that new promise becomes depends on what your callback does:

| Callback… | New promise becomes |
| --- | --- |
| returns a value `v` | fulfilled with `v` |
| returns nothing | fulfilled with `undefined` |
| returns a promise / thenable | **follows** it (waits, adopts its result) |
| throws `err` | rejected with `err` |
| isn't a function (e.g. `.then(5)`) | **ignored** — value passes through |

### Chaining

```js
fetchUser(1)
  .then((user) => fetchPosts(user.id))   // return a promise → next .then waits
  .then((posts) => posts.length)          // return a value
  .then((count) => console.log(count))
  .catch((err) => console.error(err))     // catches an error from ANY step above
  .finally(() => setLoading(false));
```

### ❌ The "forgot to return" bug

```js
fetchUser(1)
  .then((user) => {
    fetchPosts(user.id); // ❌ not returned — chain doesn't wait
  })
  .then((posts) => console.log(posts)); // undefined
```

### ❌ Nested promises (promise hell)

```js
// ❌
getUser().then((user) => {
  getPosts(user.id).then((posts) => {
    getComments(posts[0].id).then((comments) => { /* ... */ });
  });
});

// ✅ flat chain
getUser()
  .then((user) => getPosts(user.id))
  .then((posts) => getComments(posts[0].id))
  .then((comments) => { /* ... */ });
```

If you need values from earlier steps, use `async/await` (cleanest) or carry them forward.

---

# 4. Error Propagation ⭐⭐⭐

A rejection **skips** every `onFulfilled` handler until it reaches the nearest rejection handler — like `try/catch` across async steps.

```js
Promise.resolve()
  .then(() => { throw new Error("boom"); })
  .then(() => console.log("skipped"))       // skipped
  .catch((e) => {
    console.log("caught:", e.message);       // caught: boom
    return "recovered";                      // catch RECOVERS the chain
  })
  .then((v) => console.log("after:", v));    // after: recovered
```

### `.catch` recovers; re-throw to keep it rejected

```js
.catch((err) => {
  logError(err);
  throw err; // propagate further
})
```

### `.then(success, failure)` vs `.then(success).catch(failure)`

```js
// failure does NOT catch errors thrown inside success
p.then(success, failure);

// catch handles errors from p AND from success
p.then(success).catch(failure);
```

### `.finally`

- Receives **no argument**
- **Passes through** the original value/reason
- If it **throws** (or returns a rejected promise), that error **replaces** the result
- If it returns a promise, the chain **waits** for it

```js
Promise.resolve(1).finally(() => 2).then(console.log); // 1 — return value ignored
Promise.resolve(1).finally(() => { throw "x"; }).catch(console.log); // "x"
```

### Unhandled rejections

A rejected promise with no handler → `unhandledrejection` event in browsers, process crash in Node (by default since v15). Always end chains with `.catch` or use `try/catch` with `await`.

---

# 5. Static Methods ⭐⭐⭐

| Method | Resolves when | Rejects when | Result | Empty input |
| --- | --- | --- | --- | --- |
| `Promise.all` | **all** fulfill | **any** rejects (fail-fast) | array of values (input order) | `[]` immediately |
| `Promise.allSettled` | **all** settle | never | `[{status, value \| reason}]` | `[]` immediately |
| `Promise.race` | **first to settle** fulfills | **first to settle** rejects | that value/reason | **pending forever** |
| `Promise.any` | **first** fulfills | **all** reject | value / `AggregateError` | rejects with `AggregateError` |
| `Promise.resolve(v)` | — | — | promise of `v` (returns `v` itself if it's a native promise) | |
| `Promise.reject(r)` | — | — | rejected promise | |
| `Promise.withResolvers()` | — | — | `{ promise, resolve, reject }` (ES2024) | |

```js
const [user, posts] = await Promise.all([fetchUser(), fetchPosts()]);

const results = await Promise.allSettled([api1(), api2()]);
results.forEach((r) =>
  r.status === "fulfilled" ? use(r.value) : log(r.reason),
);

const fastest = await Promise.any([mirror1(), mirror2(), mirror3()]);

const data = await Promise.race([fetchData(), timeout(5000)]);
```

### Important details

- `Promise.all` **doesn't cancel** the other promises when one rejects — they keep running; their results are just ignored.
- Results keep **input order**, not completion order.
- Non-promise values are allowed: `Promise.all([1, p])` → `1` is wrapped with `Promise.resolve`.
- Choose: need everything → `all`; want partial success → `allSettled`; first success → `any`; first anything / timeout → `race`.

---

# 6. Promise Patterns

## 6.1 Sequential vs Parallel ⭐

```js
// Sequential — each waits for the previous (total = sum of times)
for (const id of ids) {
  const user = await fetchUser(id);
  results.push(user);
}

// Parallel — all start together (total ≈ slowest)
const results = await Promise.all(ids.map((id) => fetchUser(id)));
```

Use sequential only when steps **depend** on each other or must not overlap (rate limits, ordered writes).

Sequential with `reduce` (pre-async/await style):

```js
ids.reduce(
  (chain, id) => chain.then((acc) => fetchUser(id).then((u) => [...acc, u])),
  Promise.resolve([]),
);
```

## 6.2 Implement `sleep`

```js
const sleep = (ms) => new Promise((resolve) => setTimeout(resolve, ms));
```

## 6.3 Retry with exponential backoff ⭐

```js
async function retry(fn, retries = 3, delay = 500, factor = 2) {
  let attempt = 0;
  while (true) {
    try {
      return await fn(attempt);
    } catch (err) {
      if (attempt >= retries) throw err; // out of retries → give up
      await sleep(delay * factor ** attempt); // 500, 1000, 2000…
      attempt++;
    }
  }
}

// usage
const data = await retry(() => fetch("/api").then((r) => {
  if (!r.ok) throw new Error(r.status);
  return r.json();
}));
```

> `fetch` only rejects on **network** errors — a 404/500 **resolves**. Check `res.ok` and throw yourself.

Talking points: total attempts = `retries + 1`; add **jitter** (randomness) to avoid thundering herd; don't retry non-retryable errors (400, 401).

## 6.4 Timeout ⭐

```js
function withTimeout(promise, ms) {
  let timerId;
  const timeout = new Promise((_, reject) => {
    timerId = setTimeout(() => reject(new Error(`Timed out after ${ms}ms`)), ms);
  });
  return Promise.race([promise, timeout]).finally(() => clearTimeout(timerId));
}
```

⚠️ This stops **waiting**, but doesn't stop the **work**. For `fetch`, actually cancel with `AbortController`:

```js
const res = await fetch(url, { signal: AbortSignal.timeout(5000) });
```

## 6.5 Concurrency limit (promise pool) ⭐

Run many async tasks but at most `limit` at a time (e.g. upload 100 files, 3 at a time).

```js
async function promisePool(tasks, limit) {
  const results = new Array(tasks.length);
  let nextIndex = 0;

  async function worker() {
    while (nextIndex < tasks.length) {
      const i = nextIndex++;        // claim a task (safe: JS is single-threaded)
      results[i] = await tasks[i]();
    }
  }

  const workers = Array.from({ length: Math.min(limit, tasks.length) }, worker);
  await Promise.all(workers);
  return results; // input order preserved
}

// tasks are FUNCTIONS returning promises — so they don't start early
const tasks = files.map((f) => () => upload(f));
await promisePool(tasks, 3);
```

### Reusable limiter (like `p-limit`)

```js
function createLimiter(limit) {
  let active = 0;
  const queue = [];

  const next = () => {
    if (active >= limit || queue.length === 0) return;
    active++;
    const { task, resolve, reject } = queue.shift();
    Promise.resolve()
      .then(task)
      .then(resolve, reject)
      .finally(() => {
        active--;
        next();
      });
  };

  return (task) =>
    new Promise((resolve, reject) => {
      queue.push({ task, resolve, reject });
      next();
    });
}

const limit = createLimiter(2);
const results = await Promise.all(urls.map((u) => limit(() => fetch(u))));
```

## 6.6 Promise queue (strictly one at a time, tasks added any time)

```js
class PromiseQueue {
  #tail = Promise.resolve();

  add(task) {
    const result = this.#tail.then(() => task());
    this.#tail = result.catch(() => {}); // one failure shouldn't block later tasks
    return result;                        // caller still sees the real result/error
  }
}

const q = new PromiseQueue();
q.add(() => save(a));
q.add(() => save(b)); // starts only after save(a) settles
```

## 6.7 Cancellation & Race Conditions ⭐

Promises **can't be cancelled** themselves. Use `AbortController` to cancel the underlying work:

```js
const controller = new AbortController();
fetch("/api/search?q=re", { signal: controller.signal })
  .catch((err) => {
    if (err.name === "AbortError") return; // expected
    throw err;
  });

controller.abort(); // cancels the request
```

**Race condition:** user types "re" then "react"; the "re" response arrives **last** and overwrites the correct results. Fixes:
1. Abort the previous request (above), or
2. Ignore stale responses (track latest request id / `ignore` flag in `useEffect` cleanup).

Full React version in `26-react-advanced-javascript.md`.

---

# 7. Implementations ⭐⭐⭐

## 7.1 `Promise.all`

```js
function promiseAll(iterable) {
  return new Promise((resolve, reject) => {
    const items = Array.from(iterable);
    const results = new Array(items.length);
    let remaining = items.length;

    if (remaining === 0) return resolve(results); // empty → resolve immediately

    items.forEach((item, i) => {
      Promise.resolve(item).then((value) => {
        results[i] = value;            // keep input order
        if (--remaining === 0) resolve(results);
      }, reject);                      // first rejection rejects all
    });
  });
}
```

Key points: wrap with `Promise.resolve` (non-promises allowed), store by **index**, count completions (don't use `results.length` — assigning index 5 first makes length 6), handle empty input.

## 7.2 `Promise.race`

```js
function promiseRace(iterable) {
  return new Promise((resolve, reject) => {
    for (const item of iterable) {
      Promise.resolve(item).then(resolve, reject); // first settle wins; later calls ignored
    }
  });
}
```

Empty iterable → stays pending forever (matches spec).

## 7.3 `Promise.allSettled`

```js
function promiseAllSettled(iterable) {
  return promiseAll(
    Array.from(iterable, (item) =>
      Promise.resolve(item).then(
        (value) => ({ status: "fulfilled", value }),
        (reason) => ({ status: "rejected", reason }),
      ),
    ),
  );
}
```

## 7.4 `Promise.any`

```js
function promiseAny(iterable) {
  return new Promise((resolve, reject) => {
    const items = Array.from(iterable);
    const errors = new Array(items.length);
    let remaining = items.length;

    if (remaining === 0) {
      return reject(new AggregateError([], "All promises were rejected"));
    }

    items.forEach((item, i) => {
      Promise.resolve(item).then(resolve, (err) => {
        errors[i] = err;
        if (--remaining === 0) {
          reject(new AggregateError(errors, "All promises were rejected"));
        }
      });
    });
  });
}
```

`any` is the **mirror** of `all`: first fulfillment wins; all rejections → reject.

## 7.5 Promisify (callback → promise)

```js
function promisify(fn) {
  return (...args) =>
    new Promise((resolve, reject) => {
      fn(...args, (err, result) => (err ? reject(err) : resolve(result)));
    });
}
// const readFileAsync = promisify(fs.readFile);
```

## 7.6 Custom Promise (simplified, interview-level) ⭐

Covers: states, settle-once, async (microtask) callbacks, chaining, thenable adoption, errors, `catch`, `finally`.

```js
class MyPromise {
  #state = "pending";
  #value = undefined;
  #handlers = [];

  constructor(executor) {
    let done = false; // resolve/reject may only win once
    const resolve = (value) => {
      if (done) return;
      done = true;
      this.#resolveWith(value);
    };
    const reject = (reason) => {
      if (done) return;
      done = true;
      this.#settle("rejected", reason);
    };

    try {
      executor(resolve, reject);
    } catch (err) {
      reject(err); // throw in executor = rejection
    }
  }

  // Adopt the state of thenables (promises returned from .then, etc.)
  #resolveWith(value) {
    if (value === this) {
      return this.#settle("rejected", new TypeError("Chaining cycle detected"));
    }
    if (value !== null && (typeof value === "object" || typeof value === "function")) {
      let then;
      try {
        then = value.then;
      } catch (err) {
        return this.#settle("rejected", err);
      }
      if (typeof then === "function") {
        let called = false;
        try {
          then.call(
            value,
            (v) => { if (!called) { called = true; this.#resolveWith(v); } },
            (r) => { if (!called) { called = true; this.#settle("rejected", r); } },
          );
        } catch (err) {
          if (!called) { called = true; this.#settle("rejected", err); }
        }
        return;
      }
    }
    this.#settle("fulfilled", value);
  }

  #settle(state, value) {
    if (this.#state !== "pending") return;
    this.#state = state;
    this.#value = value;
    this.#handlers.forEach((h) => this.#runHandler(h));
    this.#handlers = [];
  }

  #runHandler({ onFulfilled, onRejected, resolve, reject }) {
    queueMicrotask(() => {
      const fulfilled = this.#state === "fulfilled";
      const callback = fulfilled ? onFulfilled : onRejected;

      // Non-function handler → pass value/reason through
      if (typeof callback !== "function") {
        return fulfilled ? resolve(this.#value) : reject(this.#value);
      }
      try {
        resolve(callback(this.#value)); // return value (or promise) feeds the next link
      } catch (err) {
        reject(err);
      }
    });
  }

  then(onFulfilled, onRejected) {
    return new MyPromise((resolve, reject) => {
      const handler = { onFulfilled, onRejected, resolve, reject };
      if (this.#state === "pending") this.#handlers.push(handler);
      else this.#runHandler(handler);
    });
  }

  catch(onRejected) {
    return this.then(undefined, onRejected);
  }

  finally(onFinally) {
    return this.then(
      (value) => MyPromise.resolve(onFinally()).then(() => value),
      (reason) => MyPromise.resolve(onFinally()).then(() => { throw reason; }),
    );
  }

  static resolve(value) {
    return value instanceof MyPromise ? value : new MyPromise((res) => res(value));
  }

  static reject(reason) {
    return new MyPromise((_, rej) => rej(reason));
  }
}
```

**How to present it in an interview** (build it in this order):
1. State + value + `resolve`/`reject` that settle once
2. `then` storing handlers while pending, running them when settled
3. Make `then` return a new promise → chaining
4. Run callbacks in `queueMicrotask` → async guarantee
5. Adopt returned promises/thenables
6. `catch` and `finally` on top of `then`

---

# 8. Output Questions 🧪

### Q1

```js
console.log(1);
new Promise((resolve) => {
  console.log(2);
  resolve();
  console.log(3);
}).then(() => console.log(4));
console.log(5);
```

<details><summary>Answer</summary>

`1 2 3 5 4` — executor is synchronous; `resolve()` doesn't stop execution; `.then` callback is a microtask.

</details>

### Q2

```js
Promise.resolve(1)
  .then((x) => x + 1)
  .then((x) => { x * 2; })
  .then(console.log);
```

<details><summary>Answer</summary>

`undefined` — the second callback has braces and no `return`.

</details>

### Q3

```js
Promise.resolve(1)
  .then(2)
  .then(Promise.resolve(3))
  .then(console.log);
```

<details><summary>Answer</summary>

`1` — non-function arguments to `.then` are ignored; the value passes through.

</details>

### Q4

```js
Promise.reject("err")
  .catch((e) => e + "!")
  .then((v) => console.log("then:", v));
```

<details><summary>Answer</summary>

`then: err!` — `catch` returned a value, so the chain is fulfilled.

</details>

### Q5

```js
Promise.resolve()
  .then(() => { throw "x"; }, () => console.log("handler"))
  .catch((e) => console.log("catch", e));
```

<details><summary>Answer</summary>

`catch x` — the second argument of `.then` only handles rejections of the **previous** promise, not errors thrown by the first argument.

</details>

### Q6

```js
Promise.resolve(1)
  .finally(() => 2)
  .then(console.log);

Promise.reject(new Error("a"))
  .finally(() => console.log("cleanup"))
  .catch((e) => console.log(e.message));
```

<details><summary>Answer</summary>

```text
cleanup
1
a
```

Both `finally` callbacks run in the first microtask round (order: first chain's `finally` (no log), then `cleanup`). Next round: `1`, then `a`. Key point: `finally`'s return value is ignored; the original value/reason passes through.

</details>

### Q7

```js
Promise.all([1, Promise.resolve(2), Promise.reject("no")])
  .then(console.log)
  .catch((e) => console.log("err", e));

Promise.allSettled([1, Promise.reject("no")]).then(console.log);
```

<details><summary>Answer</summary>

```text
[ { status: 'fulfilled', value: 1 }, { status: 'rejected', reason: 'no' } ]
err no
```

`Promise.all` rejects (fail-fast), but its rejection must pass through the `.then(console.log)` link before reaching `.catch` — one extra microtask hop. `allSettled`'s `.then` is directly attached, so it logs first. Content-wise: `all` → rejected with `"no"`; `allSettled` → never rejects.

</details>

### Q8

```js
Promise.any([Promise.reject(1), Promise.reject(2)])
  .catch((e) => console.log(e.constructor.name, e.errors));
```

<details><summary>Answer</summary>

`AggregateError [ 1, 2 ]`

</details>

### Q9

```js
const p = new Promise((resolve) => resolve(Promise.reject("inner")));
p.then(() => console.log("ok")).catch((e) => console.log("fail", e));
```

<details><summary>Answer</summary>

`fail inner` — resolving with a promise makes `p` **follow** it. "Resolved" doesn't mean "fulfilled".

</details>

> Ordering-heavy questions (promises mixed with `setTimeout`, `async/await`) are in `08-event-loop.md`.

---

# 9. Interview Answers

**Promise.all vs allSettled vs race vs any**

> "`all` waits for every promise and rejects as soon as one fails. `allSettled` waits for every promise and never rejects — it gives a status for each. `race` settles with whichever promise settles first, success or failure — useful for timeouts. `any` resolves with the first success and only rejects, with an `AggregateError`, if all of them fail."

**How does error propagation work?**

> "Each `.then` returns a new promise. If a callback throws or returns a rejected promise, the new promise rejects, and the rejection skips all success handlers until it reaches a `.catch` or a `.then` with a rejection handler. If that handler returns normally, the chain recovers."

**Can you cancel a promise?**

> "Not directly — a promise is just a result container. You cancel the underlying operation, for example with an `AbortController` signal for `fetch`, or you ignore stale results. `Promise.race` with a timeout only stops waiting; it doesn't stop the work."

---

# 10. Key Takeaways

- Executor runs synchronously; callbacks run as microtasks.
- Settles once; later resolve/reject ignored.
- Every `.then` returns a new promise; return values feed forward, throws reject.
- Always **return** inner promises; avoid nesting.
- `.catch` recovers; re-throw to propagate. `finally` passes the value through.
- Know `all / allSettled / race / any` behaviour including empty input.
- Implementations: wrap with `Promise.resolve`, store by index, count completions.
- `fetch` doesn't reject on HTTP errors; promises can't be cancelled — use `AbortController`.

## Readiness Checklist

- [ ] Explain promise states and the 5 rules in Section 2
- [ ] Explain what `.then` returns in every case (table in Section 3)
- [ ] Implement all / race / allSettled / any from memory
- [ ] Implement retry, timeout, concurrency pool, queue
- [ ] Build `MyPromise` step by step
- [ ] Solve all output questions
