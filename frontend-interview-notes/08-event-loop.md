# 08 — Event Loop 🔥🔥🔥

> **Interview + Understanding Guide** — `00-starter-roadmap.md` Section 8
>
> Goal: **Predict JavaScript execution order without running the code.** Explain the event loop verbally and draw its architecture.

---

# 1. The Big Picture

```text
┌─────────────────────────────── JavaScript Engine (V8) ──────────────────────┐
│                                                                              │
│   Call Stack                    Heap                                         │
│   ┌──────────┐                  (objects live here)                          │
│   │ fn()     │                                                               │
│   │ main()   │                                                               │
│   └──────────┘                                                               │
└───────│──────────────────────────────────────────────────────────────────────┘
        │ calls
        ▼
┌──────────────────── Web APIs (provided by the browser) ─────────────────────┐
│  setTimeout · setInterval · fetch/XHR · DOM events · requestAnimationFrame   │
└───────│──────────────────────────────────────────────────────────────────────┘
        │ when done, queue the callback
        ▼
┌──────────────────────────┐       ┌───────────────────────────────┐
│ Microtask Queue          │       │ Task (Macrotask) Queue        │
│ Promise .then/.catch/    │       │ setTimeout · setInterval ·    │
│ .finally · await resume  │       │ DOM events · I/O · message    │
│ queueMicrotask ·         │       │ (MessageChannel/postMessage)  │
│ MutationObserver         │       │                               │
└────────────▲─────────────┘       └──────────────▲────────────────┘
             │                                    │
             └──────────── EVENT LOOP ────────────┘
     "When the call stack is empty: run ALL microtasks,
      (maybe render), then take ONE task, repeat."
```

### Components

| Part | Role |
| --- | --- |
| **Call Stack** | Where JS code executes, one frame at a time (LIFO) |
| **Heap** | Memory for objects |
| **Web APIs** | Browser features that do work outside the JS thread (timers, network, events) |
| **Task queue** (macrotasks) | Callbacks from timers, events, I/O, messages |
| **Microtask queue** | Promise reactions, `await` continuations, `queueMicrotask`, `MutationObserver` |
| **Event loop** | Moves callbacks from the queues onto the stack when it's empty |

> The **engine** (V8) only has the stack and heap. Timers, the queues and the event loop are provided by the **host environment** (browser or Node).

---

# 2. The Algorithm ⭐⭐⭐

```text
1. Run the current script (synchronous code) until the call stack is empty
2. Run ALL microtasks
     - including microtasks queued BY microtasks (until the queue is empty)
3. If it's time to render (browser, ~every 16.7ms at 60Hz):
     - run requestAnimationFrame callbacks
     - style → layout → paint
4. Take ONE task (macrotask) from the task queue, run it
5. Go to step 2
```

### The golden rules

1. **Synchronous code first** — always, completely.
2. **Microtasks before the next macrotask** — the whole microtask queue is drained.
3. **One macrotask per loop iteration**, then microtasks again.
4. **Microtasks queued during microtasks run in the same drain** — they can't be "delayed" to after a timer.
5. Queues are **FIFO** within themselves.

---

# 3. Macrotasks vs Microtasks ⭐⭐⭐

| Macrotasks (tasks) | Microtasks |
| --- | --- |
| `setTimeout`, `setInterval` | `Promise.then / catch / finally` |
| DOM events (click, input…) | Code after `await` |
| Network/I/O callbacks | `queueMicrotask(fn)` |
| `MessageChannel`, `postMessage` | `MutationObserver` callbacks |
| `setImmediate` (Node) | `process.nextTick` (Node — runs even before promise microtasks) |
| Initial script execution | |

> `requestAnimationFrame` is neither — it runs in the **render step**, before paint.

```js
console.log("1");

setTimeout(() => console.log("2"), 0);

Promise.resolve().then(() => console.log("3"));

console.log("4");
```

Output:

```text
1
4
3
2
```

```text
Sync:        1, 4         (timer registered, promise reaction queued)
Microtasks:  3
Next task:   2
```

---

# 4. `setTimeout` — What the Delay Really Means

`setTimeout(fn, 100)` = "queue `fn` **no sooner than** 100ms from now". It's a **minimum**, not a guarantee.

```js
setTimeout(() => console.log("timeout"), 0);
const start = Date.now();
while (Date.now() - start < 1000) {} // block 1s
// "timeout" logs after ~1000ms — it had to wait for the stack
```

- `setTimeout(fn, 0)` is **not immediate** — it waits for sync code + all microtasks (+ possibly a render).
- Nested timers (≥5 levels deep) are clamped to **≥4ms** in browsers.
- **Background tabs** throttle timers to ≥1s (or more).
- Timers with smaller delays fire first; equal delays fire in registration order.

```js
setTimeout(() => console.log("A"), 100);
setTimeout(() => console.log("B"), 0);
setTimeout(() => console.log("C"), 0);
// B, C, A
```

### `setInterval` drift

`setInterval` doesn't account for how long the callback takes; if the thread is busy, ticks are delayed or skipped. For precise/animation timing use recursive `setTimeout` or `requestAnimationFrame`.

---

# 5. `async/await` and the Event Loop ⭐⭐⭐

```js
async function fn() {
  console.log("A");   // sync — runs when fn() is called
  await something;     // suspend; rest of function = microtask after `something` settles
  console.log("B");
}
```

Mental model: **everything after `await` is like a `.then` callback**.

```js
async function fn() {
  console.log("A");
  await null;
  console.log("B");
}

// ≈
function fn() {
  console.log("A");
  return Promise.resolve(null).then(() => {
    console.log("B");
  });
}
```

### The classic interview question ⭐⭐⭐

```js
async function async1() {
  console.log("async1 start");
  await async2();
  console.log("async1 end");
}

async function async2() {
  console.log("async2");
}

console.log("script start");

setTimeout(() => console.log("setTimeout"), 0);

async1();

new Promise((resolve) => {
  console.log("promise1");
  resolve();
}).then(() => console.log("promise2"));

console.log("script end");
```

Output:

```text
script start
async1 start
async2
promise1
script end
async1 end
promise2
setTimeout
```

### Step-by-step

```text
SYNC
  "script start"
  setTimeout → timer registered (task queue later)
  async1() → "async1 start"
           → async2() runs synchronously → "async2", returns resolved promise
           → await → async1 suspends; continuation queued as microtask [async1 end]
  new Promise executor (sync) → "promise1", resolve() → .then queued [async1 end, promise2]
  "script end"

MICROTASKS (drain all)
  "async1 end"
  "promise2"

TASKS
  "setTimeout"
```

---

# 6. How to Solve Any Output Question

```text
1. Go line by line. Write SYNC logs immediately.
2. Maintain two lists on paper:
     Microtasks: [ ... ]
     Tasks:      [ ... ]   (sorted by delay, then order)
3. Promise executor = SYNC. `.then` callback = MICROTASK (only once the promise is resolved).
4. Code before first `await` = SYNC; after = MICROTASK.
5. When sync code ends: run microtasks one by one; any new microtask goes to the END of the same list.
6. Microtasks empty → run ONE task → then drain microtasks again.
```

### Chained `.then` — interleaving ⭐

Each `.then` in a chain is queued only **after** the previous one runs.

```js
Promise.resolve()
  .then(() => console.log(1))
  .then(() => console.log(2));

Promise.resolve()
  .then(() => console.log(3))
  .then(() => console.log(4));
```

Output: `1 3 2 4`

```text
Microtasks: [1, 3]
run 1 → queue 2      → [3, 2]
run 3 → queue 4      → [2, 4]
run 2, run 4
```

### Returning a promise from `.then` costs extra ticks

```js
Promise.resolve()
  .then(() => {
    console.log("a");
    return Promise.resolve("b"); // adopting a promise takes 2 extra microtask hops
  })
  .then((v) => console.log(v));

Promise.resolve()
  .then(() => console.log(1))
  .then(() => console.log(2))
  .then(() => console.log(3))
  .then(() => console.log(4));
```

Output: `a 1 2 3 b 4`

> You won't be expected to derive the exact "+2 ticks" rule in most interviews, but knowing that **returning a promise is slower than returning a value** is a strong signal.

---

# 7. Microtask Starvation

Because the microtask queue is fully drained before rendering or the next task, an endless stream of microtasks **freezes the page**:

```js
function loop() {
  Promise.resolve().then(loop); // ❌ page never renders, clicks never handled
}
loop();

function loop2() {
  setTimeout(loop2, 0); // ✅ page stays responsive — browser can render between tasks
}
```

---

# 8. Rendering, `requestAnimationFrame`, `MutationObserver`

```text
Task → Microtasks → [ rAF callbacks → Style → Layout → Paint ] → Task → ...
                    └──── only when the browser decides to render (≈ each frame)
```

- **`requestAnimationFrame(cb)`** — runs `cb` right **before the next paint**. Best for animations and DOM reads/writes tied to frames. Paused in background tabs.
- DOM changes made in one task/microtask batch are painted **together** — the user never sees intermediate states.
- **`MutationObserver`** callbacks are **microtasks** — batched after DOM changes, before paint.
- **`requestIdleCallback`** — runs when the browser is idle (low-priority work).

```js
// The user never sees "red": both changes happen before paint
box.style.background = "red";
box.style.background = "blue";
```

### Splitting long work

```js
// ❌ Blocks the main thread for the whole loop
items.forEach(processHeavy);

// ✅ Yield to the browser between chunks
async function processInChunks(items, size = 100) {
  for (let i = 0; i < items.length; i += size) {
    items.slice(i, i + size).forEach(processHeavy);
    await new Promise((r) => setTimeout(r, 0)); // let render/events happen
  }
}
```

(Modern browsers also offer `scheduler.yield()` for this.)

---

# 9. Node.js Differences (brief)

Node's loop has phases: **timers → pending callbacks → poll (I/O) → check (`setImmediate`) → close callbacks**.

- `process.nextTick` queue runs **before** promise microtasks.
- `setImmediate` runs in the check phase (after I/O).
- Since Node 11, microtasks run **between each** timer callback (same as browsers).

```js
setTimeout(() => console.log("timeout"), 0);
setImmediate(() => console.log("immediate"));
Promise.resolve().then(() => console.log("promise"));
process.nextTick(() => console.log("nextTick"));
// nextTick, promise, then timeout/immediate (order of these two can vary in the main module)
```

---

# 10. Output Questions — Practice Set 🧪

Solve on paper first. Use the method in Section 6.

### Q1

```js
console.log("A");
setTimeout(() => console.log("B"), 0);
Promise.resolve().then(() => console.log("C"));
console.log("D");
```

<details><summary>Answer</summary>

`A D C B`

</details>

### Q2

```js
setTimeout(() => {
  console.log("t1");
  Promise.resolve().then(() => console.log("p-in-t1"));
}, 0);

setTimeout(() => console.log("t2"), 0);

Promise.resolve().then(() => {
  console.log("p1");
  setTimeout(() => console.log("t-in-p1"), 0);
});
```

<details><summary>Answer</summary>

`p1 t1 p-in-t1 t2 t-in-p1` — after each task, microtasks drain (`p-in-t1` before `t2`). The timer created in `p1` was registered after `t1`/`t2`.

</details>

### Q3

```js
console.log("A");
setTimeout(() => console.log("B"), 0);
Promise.resolve().then(() => {
  console.log("C");
  setTimeout(() => console.log("D"), 0);
  Promise.resolve().then(() => console.log("E"));
});
queueMicrotask(() => console.log("F"));
(async () => {
  console.log("G");
  await null;
  console.log("H");
})();
console.log("I");
```

<details><summary>Answer</summary>

`A G I C F H E B D`

```text
Sync: A, G, I          micro: [C, F, H]   tasks: [B]
C → logs C; adds D task, E micro        micro: [F, H, E]  tasks: [B, D]
F, H, E
B, D
```

</details>

### Q4

```js
const p = new Promise((resolve) => {
  console.log(1);
  setTimeout(() => {
    console.log(2);
    resolve(3);
  }, 0);
  console.log(4);
});
p.then((v) => console.log(v));
console.log(5);
```

<details><summary>Answer</summary>

`1 4 5 2 3` — the promise resolves only inside the timer, so `.then` can't run earlier.

</details>

### Q5

```js
async function a() {
  console.log("a1");
  await b();
  console.log("a2");
}
async function b() {
  console.log("b1");
  await null;
  console.log("b2");
}
a();
Promise.resolve().then(() => console.log("p"));
console.log("sync");
```

<details><summary>Answer</summary>

`a1 b1 sync b2 p a2`

```text
Sync: a1, b1 (b suspends at await → micro [b2]); a suspends awaiting b's promise
      .then queued → micro [b2, p]; "sync"
b2 → b's promise resolves → a's continuation queued → [p, a2]
p, a2
```

</details>

### Q6

```js
setTimeout(() => console.log("timeout"), 0);

Promise.resolve()
  .then(() => console.log("p1"))
  .then(() => console.log("p2"))
  .then(() => console.log("p3"));

console.log("sync");
```

<details><summary>Answer</summary>

`sync p1 p2 p3 timeout` — the entire microtask chain finishes before the timer.

</details>

### Q7

```js
console.log("start");
setTimeout(() => console.log("t100"), 100);
setTimeout(() => console.log("t0"), 0);
new Promise((res) => {
  console.log("executor");
  res();
}).then(() => console.log("then"));
console.log("end");
```

<details><summary>Answer</summary>

`start executor end then t0 t100`

</details>

### Q8

```js
for (var i = 0; i < 3; i++) {
  setTimeout(() => console.log(i), 0);
}
for (let j = 0; j < 3; j++) {
  setTimeout(() => console.log(j), 0);
}
```

<details><summary>Answer</summary>

`3 3 3 0 1 2` — `var` is shared (closure over one `i`, which is 3 when timers run); `let` creates a new binding per iteration.

</details>

### Q9

```js
Promise.resolve()
  .then(() => {
    console.log("a");
    return Promise.resolve("b");
  })
  .then((v) => console.log(v));

Promise.resolve()
  .then(() => console.log(1))
  .then(() => console.log(2))
  .then(() => console.log(3))
  .then(() => console.log(4));
```

<details><summary>Answer</summary>

`a 1 2 3 b 4` — returning a promise from `.then` adds two extra microtask hops.

</details>

### Q10

```js
async function test() {
  console.log(1);
  const x = await new Promise((r) => setTimeout(() => r(2), 0));
  console.log(x);
}
test();
setTimeout(() => console.log(3), 0);
Promise.resolve().then(() => console.log(4));
console.log(5);
```

<details><summary>Answer</summary>

`1 5 4 2 3` — the timer inside `test` was registered first, so it runs before the `3` timer; its `resolve` queues the `await` continuation, which runs before the next task.

</details>

---

# 11. Explain the Event Loop Verbally (60 seconds) ⭐

> "JavaScript runs on a single thread with one call stack. When we call async APIs like `setTimeout` or `fetch`, the browser handles them outside the JS thread. When they complete, their callbacks are placed in a queue. There are two main queues: the macrotask queue — timers, events, I/O — and the microtask queue — promise callbacks, code after `await`, `queueMicrotask`.
>
> The event loop constantly checks: if the call stack is empty, it first runs **all** microtasks, including any new ones they add. Then the browser may render. Then it takes **one** macrotask, runs it, and drains microtasks again.
>
> That's why a resolved promise's `.then` runs before a `setTimeout(fn, 0)`, why `setTimeout`'s delay is only a minimum, and why long synchronous code or infinite microtasks freeze the UI."

---

# 12. Key Takeaways

- Single thread, one call stack; host provides Web APIs, queues and the loop.
- Order: **sync → all microtasks → (render) → one macrotask → repeat**.
- Promise executor is sync; `.then` and post-`await` code are microtasks.
- Microtasks added during the drain run in the same drain → can starve rendering.
- `setTimeout(fn, 0)` is a minimum delay, not immediate.
- Chained `.then`s interleave across chains; returning a promise adds ticks.
- `rAF` runs before paint; `MutationObserver` is a microtask.
- Node adds `process.nextTick` (before promises) and `setImmediate`.

## Practice Tracker (from `00-starter-roadmap.md`)

- [ ] 10 output questions (this file)
- [ ] 20 output questions (add from `06-promises.md` + `07-async-javascript.md`)
- [ ] 30 output questions (write your own variations and verify in the browser console)
- [ ] Explain the event loop verbally in 60 seconds
- [ ] Draw the architecture diagram from memory
