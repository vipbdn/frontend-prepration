# 13 — Browser Storage

> **Interview + Understanding Guide** — `00-starter-roadmap.md` Section 13
>
> Goal: Know when to use `localStorage`, `sessionStorage`, cookies and IndexedDB, their limits, and the security implications (especially for auth tokens).

---

# 1. Overview Table ⭐

| | `localStorage` | `sessionStorage` | Cookies | IndexedDB |
| --- | --- | --- | --- | --- |
| Capacity | ~5–10 MB per origin | ~5 MB per origin | **~4 KB per cookie** | Large (hundreds of MB+, % of disk) |
| Lifetime | Until cleared | Until the **tab** closes | Until `Expires`/`Max-Age` (or session) | Until cleared |
| Scope | Origin, shared across tabs | Origin **+ tab** | Domain + path (configurable) | Origin |
| Sent to server? | ❌ | ❌ | ✅ **Automatically with every matching request** | ❌ |
| API | Sync, strings only | Sync, strings only | String (`document.cookie`) | **Async**, structured data, indexes, transactions |
| JS access | ✅ | ✅ | ✅ unless `HttpOnly` | ✅ |
| Available in Web Workers | ❌ | ❌ | ❌ | ✅ |

> "Origin" = protocol + host + port. `http://site.com` and `https://site.com` have **separate** storage.

---

# 2. `localStorage` & `sessionStorage` ⭐

Same API (the **Web Storage API**):

```js
localStorage.setItem("theme", "dark");
localStorage.getItem("theme");   // "dark"
localStorage.getItem("missing"); // null
localStorage.removeItem("theme");
localStorage.clear();
localStorage.length;
localStorage.key(0);             // name of the first key
```

### Strings only → JSON serialization ⭐

```js
localStorage.setItem("count", 5);
typeof localStorage.getItem("count"); // "string" → "5"

localStorage.setItem("user", { name: "Asha" });
localStorage.getItem("user"); // "[object Object]" ❌

// ✅
localStorage.setItem("user", JSON.stringify({ name: "Asha" }));
const user = JSON.parse(localStorage.getItem("user"));
```

### Safe wrapper (handles parse errors, quota, unavailable storage)

```js
const storage = {
  get(key, fallback = null) {
    try {
      const raw = localStorage.getItem(key);
      return raw === null ? fallback : JSON.parse(raw);
    } catch {
      return fallback; // corrupted JSON, or storage blocked (privacy mode / iframe)
    }
  },
  set(key, value) {
    try {
      localStorage.setItem(key, JSON.stringify(value));
      return true;
    } catch (err) {
      // QuotaExceededError when full
      console.warn("Storage failed:", err);
      return false;
    }
  },
  remove(key) {
    try { localStorage.removeItem(key); } catch {}
  },
};
```

### `sessionStorage` specifics

- Per **tab**: two tabs of the same site have **separate** sessionStorage.
- Survives **page reloads** and restores, but not closing the tab.
- "Duplicate tab" copies the session storage into the new tab.

Use cases: multi-step form progress, one-time UI state, scroll position for this tab.

### Limitations

- **Synchronous** → blocks the main thread; avoid storing large data or writing in hot loops.
- Strings only; ~5 MB.
- Not accessible in Web Workers / Service Workers.
- **Any JS on the page can read it** → XSS can steal anything stored there.
- Not available in some privacy modes / sandboxed iframes (throws) → wrap in `try/catch`.
- SSR: `window`/`localStorage` don't exist on the server — access only in effects / client code.

---

# 3. Storage Events (cross-tab sync)

The `storage` event fires in **other tabs** of the same origin when `localStorage` changes (not in the tab that made the change).

```js
window.addEventListener("storage", (e) => {
  if (e.key === "auth" && e.newValue === null) {
    logoutThisTab(); // user logged out in another tab
  }
  console.log(e.key, e.oldValue, e.newValue, e.url);
});
```

Use cases: sync logout/login, theme, cart across tabs. (`BroadcastChannel` is a cleaner messaging alternative.)

---

# 4. Cookies ⭐

Small key-value strings the browser **stores and sends automatically** with HTTP requests to the matching domain. Originally designed for **server** state (sessions).

### Reading/writing from JS

```js
document.cookie = "theme=dark; max-age=31536000; path=/; SameSite=Lax; Secure";
document.cookie; // "theme=dark; other=1" — ALL cookies in one string (no attributes)

// Delete = set expiry in the past
document.cookie = "theme=; max-age=0; path=/";

// Parse
const getCookie = (name) =>
  document.cookie
    .split("; ")
    .find((c) => c.startsWith(name + "="))
    ?.split("=")[1];
```

Each assignment sets **one** cookie — it doesn't overwrite the others.

Modern async alternative: `cookieStore.get/set` (Chromium-based browsers).

### Set by the server

```http
Set-Cookie: sessionId=abc123; HttpOnly; Secure; SameSite=Lax; Path=/; Max-Age=3600
```

### Cookie attributes ⭐⭐⭐ (security)

| Attribute | Meaning |
| --- | --- |
| `Expires` / `Max-Age` | Lifetime. Without it → session cookie (deleted when browser closes) |
| `Domain` | Which hosts receive it (`Domain=example.com` includes subdomains) |
| `Path` | URL path prefix it's sent for |
| `Secure` | Sent only over **HTTPS** |
| **`HttpOnly`** | **Not accessible from JavaScript** (`document.cookie` can't see it) → protects against XSS stealing it |
| **`SameSite`** | Controls sending on **cross-site** requests → CSRF protection |

### `SameSite` values

| Value | Sent on cross-site requests? |
| --- | --- |
| `Strict` | Never (even clicking a link from another site → not sent) |
| `Lax` (browser default) | Only on top-level **GET navigations** (clicking a link), not on cross-site POST/fetch/iframes |
| `None` | Always — **requires `Secure`** (used for third-party embeds) |

More on XSS/CSRF in `25-frontend-security.md`.

---

# 5. Cookies vs localStorage vs sessionStorage ⭐

| Question | Cookies | localStorage | sessionStorage |
| --- | --- | --- | --- |
| Need server to read it automatically? | ✅ | ❌ | ❌ |
| Can be hidden from JS? | ✅ `HttpOnly` | ❌ | ❌ |
| Large client-only data? | ❌ 4 KB, sent on every request (bandwidth) | ✅ | ✅ |
| Survive browser restart? | If `Max-Age` set | ✅ | ❌ |
| Shared across tabs? | ✅ | ✅ | ❌ |
| CSRF risk? | ✅ (sent automatically) — mitigate with SameSite/tokens | ❌ | ❌ |
| XSS risk? | Low if `HttpOnly` | ❗ High | ❗ High |

---

# 6. Where to Store Auth Tokens? ⭐⭐⭐ (very common question)

| Option | Pros | Cons |
| --- | --- | --- |
| **`HttpOnly; Secure; SameSite` cookie** | JS can't read it → XSS can't steal it; sent automatically | Needs CSRF protection (SameSite + CSRF token for sensitive actions); cross-domain APIs need CORS with credentials |
| `localStorage` | Simple; survives reloads; easy for `Authorization` header | **Any XSS can steal it** |
| In-memory (JS variable) | Not persisted → hard to steal at rest | Lost on reload → needs refresh mechanism |

**Common recommended approach:**

> Short-lived **access token in memory**, long-lived **refresh token in an `HttpOnly`, `Secure`, `SameSite` cookie**. On reload, call `/refresh` to get a new access token.

Interview answer:

> "I'd avoid `localStorage` for sensitive tokens because any XSS vulnerability can read it. An `HttpOnly`, `Secure`, `SameSite` cookie can't be read by JavaScript, so XSS can't exfiltrate it — though I'd still need CSRF protection. A common pattern is an in-memory access token plus a refresh token in an `HttpOnly` cookie. And XSS prevention is still essential, because XSS can make authenticated requests even if it can't steal the token."

---

# 7. IndexedDB

A **transactional, asynchronous, NoSQL database** in the browser. Stores structured data (objects, arrays, `Date`, `Blob`, `File`) using the structured clone algorithm, with indexes for querying.

```js
const request = indexedDB.open("app-db", 1);

request.onupgradeneeded = () => {
  const db = request.result;
  const store = db.createObjectStore("notes", { keyPath: "id" });
  store.createIndex("byDate", "createdAt");
};

request.onsuccess = () => {
  const db = request.result;
  const tx = db.transaction("notes", "readwrite");
  tx.objectStore("notes").put({ id: 1, text: "Hello", createdAt: new Date() });
  tx.oncomplete = () => console.log("saved");
};
```

The raw API is verbose and event-based — in practice use a wrapper like **`idb`** (promise-based) or **Dexie**.

**Use cases:** offline-first apps (with service workers), caching large API data, drafts, files/images, PWA data.

### Other storage worth knowing

- **Cache Storage API** (`caches`) — stores HTTP Request/Response pairs; used by service workers for offline.
- **`navigator.storage.estimate()`** — how much space you've used/have.
- **`navigator.storage.persist()`** — ask the browser not to evict your data.

---

# 8. Storage Limitations & Gotchas

- Quota exceeded → `QuotaExceededError` on `setItem`.
- Browsers may **evict** storage under disk pressure (except persisted). Safari may clear script-writable storage for sites not visited in 7 days (ITP).
- Private/incognito: storage cleared on close; may have smaller quotas.
- Never store secrets/PII unencrypted; users (and extensions) can read and edit everything.
- **Validate** what you read back — it could be stale, corrupted, or tampered with. Version your stored data shape.

```js
const STORAGE_VERSION = 2;
const saved = storage.get("prefs");
const prefs = saved?.version === STORAGE_VERSION ? saved : defaultPrefs;
```

---

# 9. React: Persisted State Hook

```jsx
function useLocalStorage(key, initialValue) {
  const [value, setValue] = useState(() => {
    // lazy initializer: read once, not on every render
    try {
      const raw = localStorage.getItem(key);
      return raw !== null ? JSON.parse(raw) : initialValue;
    } catch {
      return initialValue;
    }
  });

  useEffect(() => {
    try {
      localStorage.setItem(key, JSON.stringify(value));
    } catch {}
  }, [key, value]);

  // Sync across tabs
  useEffect(() => {
    const onStorage = (e) => {
      if (e.key === key) setValue(e.newValue ? JSON.parse(e.newValue) : initialValue);
    };
    window.addEventListener("storage", onStorage);
    return () => window.removeEventListener("storage", onStorage);
  }, [key]);

  return [value, setValue];
}
```

---

# 10. Output Questions 🧪

```js
localStorage.setItem("a", 1);
localStorage.setItem("b", true);
localStorage.setItem("c", null);
localStorage.setItem("d", [1, 2]);
console.log(typeof localStorage.getItem("a"));      // Q1
console.log(localStorage.getItem("b") === true);    // Q2
console.log(localStorage.getItem("c"));             // Q3
console.log(localStorage.getItem("d"));             // Q4
console.log(localStorage.getItem("zzz"));           // Q5
```

<details><summary>Answers</summary>

1. `"string"`
2. `false` — it's the string `"true"`
3. `"null"` (a string!)
4. `"1,2"`
5. `null` (actual null — key missing)

</details>

---

# 11. Key Takeaways

- localStorage = persistent, shared across tabs; sessionStorage = per tab; both sync, strings, ~5 MB, readable by any JS.
- Cookies = tiny, auto-sent to server; `HttpOnly`, `Secure`, `SameSite` are critical.
- IndexedDB = async, large, structured — for offline and big data.
- JSON-serialize + `try/catch` around storage access.
- `storage` event syncs other tabs.
- Tokens: avoid localStorage for sensitive tokens; prefer `HttpOnly` cookie (+ CSRF protection) / in-memory access token.

## Readiness Checklist

- [ ] Fill in the overview table from memory
- [ ] Explain every cookie attribute, especially `HttpOnly` and `SameSite`
- [ ] Give a confident answer to "where do you store JWTs?"
- [ ] Write a safe storage wrapper and `useLocalStorage` hook
- [ ] Explain when IndexedDB is the right choice
