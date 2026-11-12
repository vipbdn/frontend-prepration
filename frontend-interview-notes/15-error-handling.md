# 15 — Error Handling

> **Interview + Understanding Guide** — `00-starter-roadmap.md` Section 15
>
> Goal: Handle sync and async errors correctly, create custom errors, know the built-in error types, and set up global error handling for production frontend apps.

---

# 1. `try / catch / finally` ⭐

```js
try {
  const data = JSON.parse(input); // may throw SyntaxError
  render(data);
} catch (error) {
  console.error("Invalid JSON:", error.message);
  showFallback();
} finally {
  hideSpinner(); // ALWAYS runs — success, error, or even return
}
```

- `catch` binding is optional (ES2019): `try { ... } catch { ... }`
- `try` only catches errors thrown **synchronously** inside it (or awaited promises inside an `async` function).

### `finally` rules (output favourites)

```js
function f() {
  try {
    return "try";
  } finally {
    console.log("finally runs");
  }
}
f(); // logs "finally runs", returns "try"

function g() {
  try {
    return "try";
  } finally {
    return "finally"; // ❗ overrides the try's return (and swallows thrown errors)
  }
}
g(); // "finally"
```

> Never `return` from `finally` — it silently discards errors.

### `try/catch` doesn't catch async callbacks ❗

```js
try {
  setTimeout(() => {
    throw new Error("boom"); // thrown later, on a different call stack
  }, 0);
} catch (e) {
  console.log("never reached");
}
```

The callback runs after the `try` block has finished. Catch inside the callback, or use promises + `await`.

---

# 2. `throw` ⭐

You can throw **anything**, but always throw `Error` objects:

```js
throw new Error("Something failed"); // ✅ has message + stack trace
throw "Something failed";            // ❌ no stack trace, no .message
```

### Error object properties

```js
const err = new Error("Not found", { cause: originalError }); // cause: ES2022
err.name;    // "Error"
err.message; // "Not found"
err.stack;   // stack trace string (non-standard but universal)
err.cause;   // the underlying error — for wrapping
```

### Wrapping errors with `cause`

```js
try {
  await saveUser(user);
} catch (err) {
  throw new Error("Failed to save profile", { cause: err }); // keep the root cause
}
```

---

# 3. Built-in Error Types

| Type | When | Example |
| --- | --- | --- |
| `Error` | Generic base | `new Error("x")` |
| `TypeError` | Wrong type operation | `null.foo`, `undefined()`, assigning to a `const` |
| `ReferenceError` | Variable doesn't exist / TDZ | `notDefined`, using `let` before declaration |
| `SyntaxError` | Invalid code / JSON | `JSON.parse("{bad")`, `eval("if(")` |
| `RangeError` | Value out of allowed range | `new Array(-1)`, `(1).toFixed(200)`, infinite recursion |
| `URIError` | Bad URI functions | `decodeURIComponent("%")` |
| `EvalError` | Legacy | — |
| `AggregateError` | Multiple errors | `Promise.any` all rejected |
| `DOMException` | Browser APIs | `AbortError`, `QuotaExceededError`, `DataCloneError` |

```js
const user = null;
user.name;          // TypeError: Cannot read properties of null (reading 'name')

console.log(x);     // ReferenceError: x is not defined
let y; const z = 1; z = 2; // TypeError: Assignment to constant variable.

function recurse() { recurse(); }
recurse();          // RangeError: Maximum call stack size exceeded
```

### Checking error types

```js
try {
  riskyOperation();
} catch (err) {
  if (err instanceof TypeError) {
    // handle
  } else if (err.name === "AbortError") {
    // DOMException — check by name
  } else {
    throw err; // ❗ re-throw what you can't handle
  }
}
```

---

# 4. Custom Errors ⭐

```js
class AppError extends Error {
  constructor(message, options) {
    super(message, options);  // passes { cause }
    this.name = this.constructor.name; // "ValidationError" etc. for subclasses
  }
}

class ValidationError extends AppError {
  constructor(field, message) {
    super(message);
    this.field = field;
  }
}

class HttpError extends AppError {
  constructor(response) {
    super(`HTTP ${response.status}: ${response.statusText}`);
    this.status = response.status;
  }
}

try {
  throw new ValidationError("email", "Email is invalid");
} catch (err) {
  if (err instanceof ValidationError) {
    showFieldError(err.field, err.message);
  }
}
```

### Typed fetch wrapper

```js
async function request(url, options) {
  let res;
  try {
    res = await fetch(url, options);
  } catch (err) {
    if (err.name === "AbortError") throw err;
    throw new AppError("Network error — check your connection", { cause: err });
  }
  if (!res.ok) throw new HttpError(res); // fetch does NOT reject on 4xx/5xx
  return res.json();
}

try {
  const data = await request("/api/profile");
} catch (err) {
  if (err instanceof HttpError && err.status === 401) redirectToLogin();
  else showToast(err.message);
}
```

---

# 5. Promise Error Handling ⭐

```js
fetchData()
  .then(process)
  .then(render)
  .catch((err) => showError(err)) // catches errors from ANY step above
  .finally(() => setLoading(false));
```

- A throw inside `.then` → rejection → skips to next `.catch`.
- `.catch` returns normally → chain recovers. Re-throw to propagate.
- `.then(ok, fail)`: `fail` does **not** catch errors thrown in `ok`.
- A promise rejection with no handler → **unhandled rejection**.

### Throwing in a promise executor vs later

```js
new Promise(() => {
  throw new Error("caught"); // ✅ → rejection
}).catch(console.log);

new Promise(() => {
  setTimeout(() => {
    throw new Error("NOT caught"); // ❌ async throw → uncaught error, promise stays pending
  });
}).catch(console.log);
```

Details in `06-promises.md`.

---

# 6. Async/Await Error Handling ⭐

```js
async function loadProfile() {
  try {
    const user = await getUser();
    const posts = await getPosts(user.id);
    return { user, posts };
  } catch (err) {
    reportError(err);
    return null;
  }
}
```

### Patterns

```js
// Catch per call
const user = await getUser().catch(() => null);

// Tuple helper
const to = (p) => p.then((v) => [null, v], (e) => [e, null]);
const [err, user2] = await to(getUser());

// Partial failures with allSettled
const results = await Promise.allSettled([a(), b(), c()]);
const failures = results.filter((r) => r.status === "rejected");
```

### Pitfalls

- `return promise` (without `await`) inside `try` → rejection escapes the `catch`.
- `forEach(async ...)` → errors become unhandled rejections; nobody awaits them.
- Calling an async function without `await` or `.catch` → unhandled rejection.

---

# 7. Global Error Handling ⭐

Last line of defence — for logging/monitoring, not for normal control flow.

### `window.onerror` / `error` event — uncaught sync errors

```js
window.addEventListener("error", (event) => {
  // event.message, event.filename, event.lineno, event.colno, event.error
  logToService({
    message: event.message,
    stack: event.error?.stack,
    source: event.filename,
  });
});

// Older form — return true to suppress the default console error
window.onerror = (message, source, lineno, colno, error) => {
  logToService({ message, stack: error?.stack });
};
```

The `error` event (with `capture: true`) also catches **resource load failures** (broken `<img>`, `<script>`):

```js
window.addEventListener("error", (e) => {
  if (e.target instanceof HTMLImageElement) e.target.src = "/fallback.png";
}, true);
```

> Errors from cross-origin scripts appear only as `"Script error."` unless served with CORS headers and `<script crossorigin>`.

### `unhandledrejection` — promises without `.catch`

```js
window.addEventListener("unhandledrejection", (event) => {
  logToService({ reason: event.reason?.message ?? event.reason });
  // event.preventDefault(); // suppress default console warning
});
```

Node equivalents: `process.on("uncaughtException")`, `process.on("unhandledRejection")`.

### Production setup

- Use a monitoring tool (Sentry, Datadog, LogRocket) — they hook these events, attach **source maps**, user/session context, breadcrumbs.
- Show friendly messages to users; log details for developers.

---

# 8. Error Handling in React ⭐

### Error Boundaries

Catch errors thrown **during rendering**, in lifecycle methods and constructors of the **child tree**, and show a fallback UI instead of unmounting the whole app.

```jsx
class ErrorBoundary extends React.Component {
  state = { hasError: false };

  static getDerivedStateFromError() {
    return { hasError: true }; // render fallback next
  }

  componentDidCatch(error, info) {
    logToService(error, info.componentStack);
  }

  render() {
    if (this.state.hasError) return this.props.fallback ?? <p>Something went wrong.</p>;
    return this.props.children;
  }
}

<ErrorBoundary fallback={<ErrorPage />}>
  <Dashboard />
</ErrorBoundary>
```

Must be a **class component** (or use the `react-error-boundary` library).

### Error boundaries do NOT catch

- Errors in **event handlers** → use `try/catch` inside the handler
- **Async code** (`setTimeout`, promises, `fetch` callbacks) → catch and put the error in state
- Server-side rendering errors
- Errors in the boundary itself

```jsx
// Bridge async errors into the boundary
const [, setState] = useState();
const throwToBoundary = (err) => setState(() => { throw err; });

fetchData().catch(throwToBoundary);
```

Data libraries (React Query, SWR) expose `error` states and can integrate with boundaries.

---

# 9. Best Practices

- Throw `Error` objects (or subclasses), never strings.
- Catch only what you can **handle**; re-throw the rest.
- Never swallow errors silently (`catch {}`) without at least logging.
- Use `cause` when wrapping errors.
- Validate inputs early and fail fast with clear messages.
- Distinguish **expected** errors (validation, 404, offline) from **bugs** (TypeError). Show UI for the first; report the second.
- Always handle `fetch` HTTP errors (`res.ok`) and `AbortError`.
- Global handlers for logging; error boundaries for UI resilience.

---

# 10. Output Questions 🧪

### Q1

```js
function test() {
  try {
    throw new Error("a");
  } catch (e) {
    return "catch";
  } finally {
    console.log("finally");
  }
}
console.log(test());
```

<details><summary>Answer</summary>

`finally`, then `catch`.

</details>

### Q2

```js
function test() {
  try {
    throw new Error("a");
  } finally {
    return "finally";
  }
}
console.log(test());
```

<details><summary>Answer</summary>

`finally` — the error is swallowed by the `return` in `finally`.

</details>

### Q3

```js
try {
  Promise.reject(new Error("x"));
} catch (e) {
  console.log("caught");
}
console.log("after");
```

<details><summary>Answer</summary>

`after`, then an **unhandled rejection** — `try` can't catch a promise rejection unless it's `await`ed.

</details>

### Q4

```js
class MyError extends Error {}
const e = new MyError("oops");
console.log(e instanceof MyError, e instanceof Error, e.name);
```

<details><summary>Answer</summary>

`true true Error` — `name` isn't set automatically; set `this.name` in the constructor.

</details>

### Q5

```js
console.log(typeof null.x);
```

<details><summary>Answer</summary>

`TypeError: Cannot read properties of null` — `typeof` protects undeclared **variables**, not property access on `null`.

</details>

---

# 11. Interview Answers

**How do you handle errors in async code?**

> "With `async/await` I wrap awaited calls in `try/catch`, using `return await` inside `try` so rejections are caught. With promise chains I end with `.catch`. For multiple independent calls where partial failure is OK, I use `Promise.allSettled`. I also remember `fetch` doesn't reject on HTTP errors, so I check `res.ok` and throw a custom `HttpError`."

**How do you handle errors globally in a frontend app?**

> "`window` `error` events for uncaught exceptions and resource failures, `unhandledrejection` for unhandled promises, both reporting to a monitoring service with source maps. In React, error boundaries around major sections show fallback UI for rendering errors, while event handlers and async code handle their own errors."

---

# 12. Key Takeaways

- `try/catch` is synchronous (plus `await`); it can't catch errors in later callbacks.
- `finally` always runs; never `return` from it.
- Throw `Error` objects; use `cause`; subclass for custom error types (set `name`).
- Know `TypeError`, `ReferenceError`, `SyntaxError`, `RangeError`, `AggregateError`, `DOMException`.
- Promise errors: `.catch` / `try + await`; watch out for unhandled rejections.
- Global: `error`, `unhandledrejection`. React: error boundaries (not for events/async).

## Readiness Checklist

- [ ] Explain `finally` behaviour including return override
- [ ] Explain why `try/catch` doesn't catch `setTimeout` errors
- [ ] Write a custom error hierarchy and a typed `fetch` wrapper
- [ ] Set up global handlers
- [ ] Explain what error boundaries do and don't catch
