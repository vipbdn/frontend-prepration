# 27 — Security in React Apps

> **Interview + Understanding Guide** — Roadmap Section 27
>
> Goal: Know React-specific security risks (XSS vectors React doesn't protect against, unsafe URLs, SSR injection), authentication and token storage, CSRF, secrets in bundles, dependency risk, and Server Component/Action security.
>
> Background on XSS/CSRF/CORS/CSP in general: see the JavaScript notes `25-frontend-security.md`.

---

# 1. What React Protects You From

React **escapes** all values rendered with `{}`:

```jsx
const comment = '<img src=x onerror="alert(document.cookie)">';
<p>{comment}</p>; // rendered as text ✅
```

React also sets **properties safely** and its elements carry an internal `$$typeof: Symbol(react.element)` marker, so JSON from a server can't masquerade as a React element (symbols can't be serialized in JSON).

**But React can't protect you from everything below.**

---

# 2. XSS Vectors in React ⭐

### 2.1 `dangerouslySetInnerHTML`

```jsx
// ❌ Untrusted HTML
<div dangerouslySetInnerHTML={{ __html: userBio }} />

// ✅ Sanitize first
import DOMPurify from "dompurify";
<div dangerouslySetInnerHTML={{ __html: DOMPurify.sanitize(userBio) }} />
```

Use for CMS content, Markdown output, rich-text editors — always sanitize (DOMPurify; on the server: `isomorphic-dompurify` or sanitize at write time). Prefer rendering Markdown to React elements (`react-markdown` without raw HTML) instead.

### 2.2 `javascript:` URLs ⭐

```jsx
<a href={user.website}>Website</a> // user.website = "javascript:alert(document.cookie)"
```

React warns (and future versions block) `javascript:` URLs, but **validate anyway**:

```jsx
function safeUrl(url) {
  try {
    const u = new URL(url, window.location.origin);
    return ["http:", "https:", "mailto:", "tel:"].includes(u.protocol) ? u.href : "about:blank";
  } catch {
    return "about:blank";
  }
}
<a href={safeUrl(user.website)} target="_blank" rel="noopener noreferrer">Website</a>
```

Same for `src` of `iframe`, `formAction`, `window.location = …`, `window.open(...)`.

### 2.3 Direct DOM manipulation

```jsx
ref.current.innerHTML = userInput;          // ❌ bypasses React escaping
document.write(...);                        // ❌
eval(userInput); new Function(userInput);   // ❌
```

### 2.4 Spreading untrusted props

```jsx
<div {...userControlledObject} />  // ❌ could inject dangerouslySetInnerHTML, href, event handlers
```

Whitelist props explicitly.

### 2.5 SSR state injection

```jsx
// ❌ A string containing "</script><script>alert(1)</script>" breaks out
<script dangerouslySetInnerHTML={{ __html: `window.__STATE__ = ${JSON.stringify(state)}` }} />

// ✅ Escape <, >, & and line separators
const safeJson = JSON.stringify(state).replace(/</g, "\\u003c").replace(/ | /g, "");
```

(Use `serialize-javascript` or the framework's mechanism.)

### 2.6 Third-party scripts & libraries

Analytics, chat widgets, ad scripts run with full access to your page. Load only trusted scripts, use **Subresource Integrity** (`integrity`) for CDN files, and a **Content Security Policy**.

---

# 3. Content Security Policy

Defence in depth against XSS — restrict where scripts can come from.

```http
Content-Security-Policy: default-src 'self'; script-src 'self' 'nonce-abc123'; object-src 'none';
  base-uri 'self'; frame-ancestors 'none'; connect-src 'self' https://api.example.com
```

- Avoid `'unsafe-inline'` / `'unsafe-eval'` for scripts.
- Next.js supports **nonce-based CSP** via middleware.
- Runtime CSS-in-JS may need `style-src` adjustments.

---

# 4. Authentication & Token Storage ⭐⭐⭐

| Storage | XSS can steal? | CSRF risk? | Notes |
| --- | --- | --- | --- |
| `localStorage` / `sessionStorage` | ✅ **yes** | ❌ no | Simple, but any XSS = token theft |
| JS memory (variable) | Hard (gone on reload) | ❌ | Needs silent refresh |
| **HttpOnly, Secure, SameSite cookie** | ❌ no | ✅ (mitigated by SameSite + CSRF tokens) | Recommended for sessions/refresh tokens |

**Recommended pattern:**
- Session ID or **refresh token in an `HttpOnly; Secure; SameSite=Lax/Strict` cookie** set by the server.
- Short-lived access token in memory (if needed for APIs on other domains), refreshed via the cookie.
- Or a **BFF** (Backend-for-Frontend): the frontend only talks to its own server with a session cookie; the BFF holds third-party tokens.

> Even with HttpOnly cookies, XSS can still **make requests as the user** while they're on the page — XSS prevention remains essential.

### Auth UX in React

```jsx
function useAuth() { /* user, login, logout, isLoading via context + /me endpoint */ }

// On 401 from API: try refresh once, else logout and redirect
api.interceptors.response.use(undefined, async (error) => {
  if (error.response?.status === 401 && !error.config._retry) {
    error.config._retry = true;
    await api.post("/auth/refresh");     // cookie-based
    return api(error.config);
  }
  if (error.response?.status === 401) logout();
  return Promise.reject(error);
});
```

- Don't decode JWTs to **authorize** on the client — only to display info. The server authorizes.
- Clear caches on logout (`queryClient.clear()`, reset stores).
- **Client-side route guards are UX**, not security.

---

# 5. CSRF

Cookie-based auth is vulnerable to cross-site request forgery.
- `SameSite=Lax` (browser default) blocks most cross-site POSTs.
- Add **CSRF tokens** (double-submit cookie or synchronizer token) for state-changing requests.
- Never change state with GET.
- Check `Origin` header server-side.
- Server Actions in Next.js compare `Origin` and `Host` headers and use POST only.

---

# 6. Secrets & Environment Variables ⭐

**Everything in your frontend bundle is public.**

- `VITE_*`, `NEXT_PUBLIC_*`, `REACT_APP_*` variables are **inlined into client JS**.
- Never put API secrets, DB credentials, private keys, or admin tokens in them.
- Call third-party APIs that need secrets **through your server** (route handlers, Server Actions, BFF).
- Use `import "server-only"` in modules that touch secrets (Next.js) so they can't be bundled for the client.
- Source maps in production can expose original code — upload them privately to your error tracker instead of serving them.

---

# 7. Server Components & Server Actions Security ⭐

- **Server Actions are public HTTP endpoints.** Anyone can call them with any arguments.
  - Validate input (Zod).
  - Authenticate **and** authorize inside every action (don't trust that the UI only shows the button to admins).
  - Don't accept IDs for ownership checks without verifying (`where: { id, userId: session.userId }`).
- **Data passed from Server to Client Components is sent to the browser** — don't pass whole DB records with sensitive fields (password hashes, internal notes). Use DTOs / a data access layer that returns only safe fields.
- React's **taint APIs** (`experimental_taintObjectReference`, `taintUniqueValue`) can prevent specific objects/values from being passed to the client.
- Keep the React/Next.js version patched — RSC protocol vulnerabilities have been disclosed and fixed (e.g. the critical "React2Shell" RCE in RSC packages, Dec 2025); upgrade promptly when advisories appear.

---

# 8. Dependency & Supply-Chain Security

- `npm audit`, Dependabot/Renovate, Snyk/Socket.
- Pin versions with lockfiles; review new dependencies (popularity, maintenance, install scripts).
- Watch for typosquatting and compromised packages (several large npm supply-chain incidents in 2025).
- Minimize dependencies — fewer packages, smaller attack surface.

---

# 9. Other Frontend Concerns

| Issue | Mitigation |
| --- | --- |
| Clickjacking | `frame-ancestors 'none'` / `X-Frame-Options: DENY` |
| Open redirects (`?next=https://evil.com`) | Allow only relative paths / allowlisted hosts |
| `target="_blank"` | `rel="noopener noreferrer"` (implied `noopener` in modern browsers) |
| Sensitive data in URLs | Avoid tokens/PII in query strings (logged, shared, referrer) |
| Logging PII to console/analytics | Scrub before sending; disable verbose logs in prod |
| File uploads | Validate type/size client-side for UX, server-side for security |
| Prototype pollution | Avoid unsafe deep merges of user JSON |
| Rate limiting / brute force | Server-side; show CAPTCHA after failures |

---

# 10. Interview Answers

**Is React safe from XSS?**

> "React escapes everything rendered through JSX expressions, which prevents most injection. But it doesn't protect `dangerouslySetInnerHTML`, `javascript:` URLs in `href`/`src`, direct DOM manipulation through refs, `eval`, spreading untrusted props, or unsafely serialized SSR state. I sanitize HTML with DOMPurify, validate URL protocols, avoid raw DOM writes, and add a CSP as defence in depth."

**Where do you store auth tokens?**

> "Preferably in an HttpOnly, Secure, SameSite cookie set by the server so JavaScript — and therefore XSS — can't read it, with CSRF protection for state-changing requests. If an access token is needed client-side, keep it short-lived in memory and refresh via the cookie. I avoid localStorage for sensitive tokens."

**How do you secure Server Actions?**

> "Treat them as public API endpoints: validate the input with a schema, check the session and authorize the specific resource inside the action, and return only the data the client needs. Never rely on the UI hiding a button."

---

# 11. Key Takeaways

- JSX escaping covers text; dangerous sinks remain: `dangerouslySetInnerHTML`, URLs, refs/innerHTML, eval, prop spreading, SSR JSON.
- Sanitize HTML; validate URL protocols; CSP + SRI.
- Tokens: HttpOnly Secure SameSite cookies > localStorage; CSRF protection; server authorizes.
- Public env vars are public; secrets stay on the server; `server-only`.
- Server Actions = public endpoints: validate, authenticate, authorize, minimize returned data.
- Keep dependencies and frameworks patched.

## Readiness Checklist

- [ ] List 6 XSS vectors React doesn't prevent and their fixes
- [ ] Write a `safeUrl` helper
- [ ] Defend a token-storage choice
- [ ] Secure a Server Action end to end
- [ ] Explain what's exposed by `NEXT_PUBLIC_` / `VITE_` variables
